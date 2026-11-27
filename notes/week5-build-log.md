# Week 5 Build Log: Load Testing to 1,000 Concurrent

Dates: 2026-11-23 to 2026-11-25. Pi as the control host, engines and load client on Modal.
Code and results: [`inference-engineering/benchmarks/`](https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks) and [`observability/`](https://github.com/crypticseeds/inference-engineering/tree/main/observability).

## What I Built

- **Matched launchers** for vLLM and SGLang with one layout and a `GPU_PROFILE` switch: `A10` (eager, 16 running requests, comparable with earlier runs) and `H100` (CUDA graphs, 256 running requests).
- **A Modal-hosted GuideLLM client** (`guidellm_modal.py`) with named plans, pinned to the GPUs' region. It waits for `/health`, writes reports to a Modal Volume, and leaves the Pi only to collect results.
- **One Grafana dashboard for both engines** on the same axes, 12 panels: running and queued requests, mean and p95 server TTFT, mean queue time, request, prompt and output token throughput, p95 end-to-end and inter-token latency, KV cache usage and preemptions.
- A Prometheus config template filled by a script, so live hostnames never land in the repo.

## Versions

| Component | Version |
|---|---|
| vLLM | 0.21.0, `--enable-prefix-caching --enable-prompt-tokens-details` |
| SGLang | 0.5.9, RadixAttention (default) plus `--enable-cache-report --enable-metrics` |
| Model | `Qwen/Qwen3-8B` @ `b968826`, BF16, served as `qwen3-8b`, 8,192-token context |
| GPUs | Modal A10 (24 GB) and H100, us-east, one per engine |
| Memory | vLLM `--gpu-memory-utilization 0.90`, SGLang `--mem-fraction-static 0.85` |
| GuideLLM | 0.8.1 |
| Workload | Streaming `/v1/chat/completions`, synthetic text, seed 42, 64 configured prompt tokens (72 after the chat template), 16 output tokens |
| Prometheus / Grafana | `v3.5.0` / `12.1.0`, 5 s scrape |

## Runs

| Day | Run | Client | Result |
|---|---|---|---|
| Mon | Smoke, 10-request sequential baseline, 1 and 2 req/s on vLLM A10 | Pi | 155.9 ms p50 TTFT, 33.6 ms p50 ITL sequential |
| Tue | Rate sweep 1 to 32 req/s on vLLM, then SGLang (with and without `/metrics`) | Pi | Under 500 ms p95 TTFT through 16 req/s on vLLM, 8 req/s on SGLang |
| Wed | `vllm bench serve` at 16 and 32 req/s, loopback in the container | In-container | 128.8 ms p95 TTFT at 16, 2.44 s at 32. Same verdict as GuideLLM |
| Thu | Same A10 sweep, both engines at once, live dashboard | Modal, us-east | Capacity unchanged. SGLang low-load p95 TTFT 268 to 77 ms with the client next to the GPU |
| Fri | H100 dry run (10, 250 concurrent, 30 s), then 1 to 1,000 concurrent, 60 s stages | Modal, 16 CPUs | vLLM peak 327 req/s at 500; SGLang 185 req/s at 100. Zero errors, zero preemptions |

### A10 at 16 and 32 req/s (Thursday, live run)

| | vLLM | SGLang |
|---|---:|---:|
| Completed req/s at 16 | 16.02 | 16.02 |
| p95 TTFT at 16 | 244 ms | 833 ms |
| Completed req/s at 32 | 20.80 | 17.44 |
| p95 TTFT at 32 | 2,482 ms | 3,142 ms |
| Queue peak at 32 | 46 | 60 |
| Mean queue time at 32 | about 2.1 s | about 2.7 s |

### H100 concurrency sweep (Friday)

| Concurrent | vLLM / SGLang req/s | vLLM / SGLang p95 TTFT (ms) | vLLM / SGLang mean queue time | vLLM / SGLang mean server TTFT |
|---:|---:|---:|---:|---:|
| 1 | 6.9 / 6.5 | 44 / 52 | 0.02 / 0.2 ms | 17 / 26 ms |
| 100 | 247.1 / 184.8 | 277 / 641 | 0.07 / 2.4 ms | 165 / 151 ms |
| 250 | 280.4 / 176.2 | 602 / 1,234 | 0.13 / 4.4 ms | 359 / 445 ms |
| 500 | 326.6 / 168.2 | 1,303 / 3,083 | 104 / 8 ms | 909 / 1,799 ms |
| 1,000 | 306.8 / 143.4 | 3,328 / 11,137 | 83 / 22 ms | 2,503 / 4,806 ms |

KV cache peaked at 5.8% (vLLM) and 4.9% (SGLang). Full tables: [A10 live run](https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks/2026-11-25-modal-a10-live), [H100 run](https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks/2026-11-25-modal-h100-concurrency), [Monday to Wednesday runs](https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks/2026-11-23-modal-a10-guidellm).

## Setup Problems

| Problem | Fix |
|---|---|
| GuideLLM exited on HTTP 503 while the engine was still loading | Client waits for `/health` before the first stage |
| SGLang `/metrics` returned 404 | `--enable-metrics` |
| Modal proxy auth blocked Prometheus | Unauthenticated `modal serve` endpoints, host replaced by `MODAL_HOST` in every committed file, scrape config rendered outside the repo |
| GenAI-Perf deprecated, SDK image 7.5 GiB | GuideLLM `concurrent` profile instead (see [note](week5-perf-analyzer-genai-perf.md)) |
| Pi client too far from the GPUs and too weak for high load | GuideLLM in a Modal CPU container in us-east; 16 CPUs for the H100 plan |
| vLLM's TTFT histogram has no bucket between 0.1 and 0.25 s | Dashboard shows mean TTFT (sum / count) next to p95 |
| `modal volume get` failed on DNS for Modal's storage host, leaving empty files | Re-downloaded once DNS resolved |
| H100 reports over GitHub's file size limit | One gzip-compressed JSON per stage |
| `docker compose` plugin not on the path | Called the compose plugin binary directly |

## Findings

1. **On the A10, the scheduler queue explains the latency.** Both engines filled their 16-request batch at 32 req/s, and mean queue time (2.1 s vLLM, 2.7 s SGLang) was most of the server-side TTFT.
2. **On the H100, it does not.** At 1,000 concurrent, vLLM's mean queue time was about 0.1 s against 2.5 s mean server TTFT, and SGLang never filled its batch. The engines' own counters held at most about 384 (vLLM) and 263 (SGLang) of 1,000 in-flight requests. The rest were somewhere before the scheduler: Modal's proxy, the HTTP server, or tokenization. Not yet established which.
3. **vLLM's throughput stopped where its batch filled.** 256 running from 500 concurrent, peak 327 req/s. That is the `--max-num-seqs` plateau the [tuning article](week5-vllm-tuning-strategies.md) describes.
4. **Memory was never the limit.** 88 tokens per request kept KV cache under 6%, so no preemptions. This workload tests scheduling and request handling, not memory.
5. **Client location changes client TTFT by engine.** Moving the client next to the GPUs cut SGLang's low-load TTFT by about 190 ms and vLLM's by under 10 ms on the A10. On the H100 the engines were within 7 ms at low load.
6. **The client can be the bottleneck.** At the A10's 32 req/s stage, GuideLLM hit its 64-request cap with 18.7 to 24.1 s p95 dispatch delay. That stage measured saturation, not 32 req/s.
7. **Concurrent clients drift.** Two clients started together were 10 to 40 s apart per stage by the end. Same load, offset time axes.

## What Surprised Me

- That past 250 concurrent on the H100, most of the time to first token was spent before either engine's scheduler, not in its queue. The dashboard showed an empty or barely used queue while users waited seconds.
- How much the "same" A10 numbers depended on where the client sat, and that it affected the two engines so differently.
- That the whole two-engine A10 and H100 session cost about 4 USD of GPU time.

## Next

- Week 5 buffer: the pre-publish benchmark checklist. Done 2026-11-27; see the [audit](https://github.com/crypticseeds/inference-engineering/blob/main/benchmarks/pre-publish-checklist.md).
- Week 4 buffer: the shared-prefix TTFT comparison. Done 2026-11-27; see the [Week 4 build log](week4-build-log.md#shared-prefix-comparison-buffer-run-2026-11-27).
- Week 6: quantization.
