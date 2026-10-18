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
