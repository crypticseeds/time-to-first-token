# Socials notes

Draft posts for LinkedIn / X at key stages of the roadmap. Post findings, not activity.

## Planned posting moments

- [ ] Week 1 end - kickoff: the 10-week build + why decode is memory-bound
- [ ] Week 2 - endpoint live + PagedAttention explained simply
- [ ] Week 3 - Grafana dashboard screenshot (draft below)
- [ ] Week 5 - the 1000-concurrent load test result
- [ ] Week 6/7 - a tradeoff finding (INT4 quality cost, spec decode crossover)
- [ ] Week 10 - the published benchmark writeup

## Drafts

### Week 2: vLLM endpoint live

#### LinkedIn

Week 2 of my LLM inference engineering roadmap is complete.

This week I moved from theory to a working service: Qwen3-8B running behind vLLM on a Modal A10 GPU, exposed through an OpenAI-compatible API and tested through health, model discovery, and chat-completion requests.

The most useful concept was PagedAttention. Instead of reserving one large contiguous Key-Value (KV) cache region for each request, vLLM divides the cache into fixed-size blocks. A block table maps each request's logical token blocks to physical GPU blocks, allowing memory to be allocated as needed and reducing fragmentation.

I also followed the request path through the vLLM V1 scheduler, KV Cache Manager, Block Pool, scheduler output, and worker-side block table. Seeing that path in source code made the architecture much clearer than reading the high-level description alone.

No performance claims yet. The next step is observability: measuring Time to First Token (TTFT), Time Per Output Token (TPOT), throughput, and queue depth before attempting optimizations.

Implementation: https://github.com/crypticseeds/inference-engineering

#LLM #vLLM #MLOps #InferenceEngineering #AIInfrastructure

#### X

Week 2 of my LLM inference engineering roadmap is complete.

I deployed Qwen3-8B with vLLM on a Modal A10 and verified its OpenAI-compatible API.

Key lesson: PagedAttention stores the KV cache in fixed-size blocks, then uses a block table to map logical token blocks to physical GPU memory. This reduces fragmentation and lets requests grow without one large contiguous allocation.

Next: TTFT, TPOT, throughput, and queue-depth observability.

https://github.com/crypticseeds/inference-engineering

### Week 3: measurement dashboard

Attach: the Grafana screenshot from the ramp (all 13 panels).

#### LinkedIn

Week 3 of my LLM inference engineering roadmap: measurement before optimization.

I put Prometheus and Grafana in front of Qwen3-8B on vLLM (one Modal A10) and drove it with an open-loop ramp from my own load generator, llm-slo-bench. 300 requests, 0 errors.

Three things the numbers taught me:

1. Two tools, two "TTFTs". vLLM measured a 94.5 ms mean time to first token. My client measured 195 ms. Same requests; the 100 ms difference is the network and TLS between my machine and the GPU. Only one of those is what a user feels, so a benchmark has to say which one it reports.

2. The network does not touch inter-token latency. Both sides saw about 36.6 ms between tokens. That is roughly 73% of the A10's memory-bandwidth limit for 16 GB of BF16 weights: decode is memory-bound, exactly as the roofline says.

3. A dashboard percentile is only as good as its histogram buckets. The stock vLLM dashboard showed a p99 queue time of 297 ms on an idle server. The real mean was 7.8 microseconds. vLLM's smallest queue-time bucket is 0.3 s, so Grafana was interpolating inside one bucket. Three of the fourteen example panels had no per-request meaning; I fixed them and wrote a definition for every panel that stayed.

No capacity claims yet: the server peaked at 5 running requests and an empty queue. Week 5 pushes it to saturation.

Dashboard and configs: https://github.com/crypticseeds/inference-engineering/tree/main/observability
Load generator: https://github.com/crypticseeds/llm-slo-bench

#LLM #vLLM #Observability #InferenceEngineering #SRE

#### X

Week 3 of my LLM inference roadmap: Prometheus + Grafana on Qwen3-8B / vLLM / one A10.

- Server TTFT 94.5 ms, client TTFT 195 ms. The gap is network. Say which one you report.
- ITL 36.6 ms both sides: about 73% of A10 memory bandwidth.
- Stock dashboard showed p99 queue time 297 ms. Real mean: 7.8 µs. Histogram buckets.

https://github.com/crypticseeds/inference-engineering/tree/main/observability

### Week 5: 1,000 concurrent requests

Attach: `grafana-h100-sweep.png` (full sweep, both engines) and `grafana-h100-500-1000.png` (the saturation stages) from the H100 run directory.

#### LinkedIn

Week 5 of my LLM inference engineering roadmap: load testing to 1,000 concurrent requests.

Same model (Qwen3-8B, BF16, pinned revision), same workload, two engines side by side: vLLM 0.21 and SGLang 0.5.9, each on its own H100 with CUDA graphs and a 256-request batch limit. GuideLLM drove a concurrency sweep from 1 to 1,000 streaming requests from a client in the same region, while Prometheus and Grafana watched both servers live.

Both engines finished every stage with zero errors and zero preemptions. vLLM peaked at 327 requests/s at 500 concurrent; SGLang at 185 requests/s at 100. Against a p95 time-to-first-token target of 500 ms, vLLM held up to 100 concurrent and SGLang to 50.

Three things the numbers taught me:

1. Throughput stops where the batch fills. vLLM's 256 running slots filled at 500 concurrent, and that is exactly where throughput flattened. Past that point, more users only add waiting time. That is the knee you set `--max-num-seqs` from.

2. The queue you can see is not always where the time goes. On a small A10, the scheduler queue explained the latency: about 2 seconds of queueing at saturation. On the H100 at 1,000 concurrent, vLLM's mean queue time was 0.1 s while its time to first token was 2.5 s, and SGLang barely queued at all. Most requests were waiting before either engine's scheduler. Finding out where is next.

3. Check the load generator before trusting the result. On the A10, my client could not keep up at 32 requests/s and sent requests up to 24 s late. That stage measured saturation, not 32 requests/s. Moving the client next to the GPUs also changed one engine's low-load latency by 190 ms and the other's by under 10 ms.

Memory never mattered here: with 88-token requests the KV cache stayed under 6%. Long prompts are a different test.

Total GPU cost for the A10 and H100 sessions: about 4 USD.

Benchmarks and dashboards: https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks

#LLM #vLLM #SGLang #LoadTesting #InferenceEngineering

#### X

Week 5 of my LLM inference roadmap: vLLM vs SGLang to 1,000 concurrent on H100s (Qwen3-8B, GuideLLM, live Grafana).

- vLLM peak 327 req/s at 500 concurrent, SGLang 185 at 100. Zero errors.
- Throughput flattened exactly when vLLM's 256-slot batch filled.
- At 1,000 concurrent, queue time 0.1 s, TTFT 2.5 s. The wait was before the scheduler.

https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks
