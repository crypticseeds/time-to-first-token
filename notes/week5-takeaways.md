# Week 5 Takeaways: Load Testing to 1,000 Concurrent

2026-11-25. All five weekday sessions are done. vLLM and SGLang both completed a concurrency sweep to 1,000 concurrent requests on Modal H100s with zero request errors, watched live in Grafana. The pre-publish checklist is still open. [Companion benchmarks](https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks)

## What the Week Was About

A benchmark number is only defensible when the workload, the client and the server are pinned and recorded. Every run this week used `Qwen/Qwen3-8B` at a pinned revision in BF16, a streaming `/v1/chat/completions` workload with fixed lengths (72 prompt tokens after chat formatting, 16 output tokens), seed 42, and the exact engine flags in the launch files. Fixed lengths keep the KV cache per request constant, so changes in latency come from load, not from the prompts.

## What I Ran

| Day | What | Result |
|---|---|---|
| Mon | Installed GuideLLM 0.8.1, smoke test and sequential baseline against vLLM on an A10 | 155.9 ms median TTFT, 33.6 ms median inter-token latency, single client |
| Tue | Fixed-workload rate sweep, 1 to 32 req/s, vLLM then SGLang on A10s | Under 500 ms p95 TTFT through 16 req/s on vLLM and 8 req/s on SGLang. Both hit their 16-request batch limit at 32 req/s |
| Wed | `vllm bench serve` cross-check at 16 and 32 req/s | Same verdict as GuideLLM: 128.8 ms p95 TTFT at 16 req/s, 2.44 s at 32 req/s |
| Thu | Repeated the A10 sweep on both engines at once, client moved to Modal, live Grafana | Capacity unchanged. SGLang's low-load client TTFT fell from 268 ms to 77 ms p95 once the client sat next to the GPU |
| Fri | H100 concurrency sweep from 1 to 1,000 concurrent, CUDA graphs on, 256-request batch | vLLM peaked at 327 req/s at 500 concurrent, SGLang at 185 req/s at 100. Zero errors and zero preemptions on both |

Thursday's roadmap item named genai-perf. I used GuideLLM's concurrent profile instead, after deciding the 7.5 GiB NVIDIA SDK image was too large for the Pi. The cross-check against a second tool is `vllm bench serve` from Wednesday, which covers the A10 rate sweep but not the H100 concurrency sweep.

## What I Learned

**Where the knee is depends on the GPU and the engine limits.** On the A10 in eager mode, both engines saturated at their 16-request batch, and the scheduler queue explained the latency: at 32 req/s, mean queue time was 2.1 s on vLLM and 2.7 s on SGLang. On the H100 the knee moved to 100 to 500 concurrent, and the scheduler queue no longer explained it. At 1,000 concurrent, vLLM's mean queue time was 0.1 s against 2.5 s of server-side TTFT, and SGLang never filled its batch. Most requests were waiting somewhere before the scheduler: Modal's proxy, the HTTP server, or tokenization.

**Memory was never the limit.** KV cache usage stayed under 6% on every run, because each request held only 88 tokens. This workload stresses scheduling and request handling, not memory. Preemption, the thing the Friday session said to watch for, needs longer prompts or outputs to show up.

**Client placement is part of the result.** Moving GuideLLM from the Pi to a Modal container in the GPUs' region cut SGLang's low-load TTFT by about 190 ms but vLLM's by under 10 ms. On the H100, with CUDA graphs, the two engines were within 7 ms at low load. Client TTFT is only comparable between runs from the same client location.

**Bucket-limited percentiles mislead at low load.** vLLM's TTFT histogram has no bucket between 0.1 and 0.25 s, so `histogram_quantile` reported about 0.25 s p95 while the true mean was under 0.1 s. The dashboard now shows the mean from sum / count beside the p95.

**The client can be the bottleneck.** At the A10's 32 req/s stage, GuideLLM hit its 64-request cap and sent late, with up to 24 s p95 dispatch delay. That stage measured saturation, not 32 req/s. On the H100, a 16-CPU client kept vLLM's dispatch delay under GuideLLM's 0.2 s threshold, but some SGLang requests still started late at 250 and 1,000 concurrent.

## What I Built

- vLLM and SGLang Modal launchers with one shared layout and a `GPU_PROFILE` switch between the A10 and H100 configurations.
- A Prometheus and Grafana stack with one dashboard that plots both engines on the same axes: batch state, mean and p95 TTFT, queue time, throughput, KV cache and preemptions.
- A Modal-hosted GuideLLM runner, so the Pi only collects results.

Total GPU spend for the Thursday and Friday runs was about 4 USD.

## Open Items

- Pre-publish benchmark checklist (Week 5 buffer).
- The shared-prefix TTFT comparison deferred from Week 4. The engines and client are now ready for it.
- Why about 620 (vLLM) to 740 (SGLang) of the 1,000 in-flight H100 requests never reached either engine's scheduler. Per-stage timing inside the server would answer it.

[A10 live run](https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks/2026-11-25-modal-a10-live) · [H100 concurrency run](https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks/2026-11-25-modal-h100-concurrency) · [Week 4 notes](week4-takeaways.md)
