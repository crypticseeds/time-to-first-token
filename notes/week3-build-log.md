# Week 3 Build Log: Measurement Dashboard

Date: 2026-10-18. Done in one session on the Pi (`agents-pi`) against the Modal endpoint.
Code and results: [`inference-engineering/observability/`](https://github.com/crypticseeds/inference-engineering/tree/main/observability).

## What I Built

- Prometheus scraping vLLM's `/metrics` on Modal over HTTPS every 5 s, Grafana with the datasource and dashboard provisioned from files.
- vLLM's example dashboard, with four panels fixed or replaced (below) and every remaining panel mapped to a written definition.
- Two load sources: `trickle.sh` (curl loop) and a [llm-slo-bench](https://github.com/crypticseeds/llm-slo-bench) open-loop ramp.

## Versions

| Component | Version |
|---|---|
| vLLM | 0.21.0, `Qwen/Qwen3-8B` @ `b968826`, BF16, served as `qwen3-8b` |
| GPU | 1x NVIDIA A10 (24 GB) on Modal, `max_containers=1` |
| vLLM flags | `--max-model-len 8192 --max-num-seqs 16 --gpu-memory-utilization 0.90 --enforce-eager` |
| Prometheus / Grafana | `v3.5.0` / `12.1.0` |
| Dashboard source | `vllm-project/vllm` `examples/observability/prometheus_grafana` @ `79e205e` |
| llm-slo-bench | `d94f17f` |
| Load generator host | Raspberry Pi (arm64) -> Modal us-east over the public internet |

## Setup Problems

| Problem | Fix |
|---|---|
| vLLM's Prometheus/Grafana docs page is a 404 | Took the files from the GitHub example folder, pinned to a commit |
| Example scrapes `host.docker.internal:8000` | `scheme: https`, target `<modal-host>:443` |
| Port 3000 taken on the Pi | Grafana on host port 3001 |
| Browser couldn't resolve `agents-pi` (an SSH alias, not DNS) | SSH port forward / real hostname |
| llm-slo-bench rejects `reserve_cost_per_request_usd: 0` | Set to 0.001; caught by a validation dry-run before GPU time |
| Prometheus scrapes keep the A10 from scaling to zero | `docker compose down` at the end of every session |

## Dashboard Fixes

Of the 14 example panels, three were wrong or undefined:

1. **Queue Time** and **Prefill/Decode Time** plotted `rate(_sum)` alone. That is seconds of queueing per second across the server, not time per request. Changed to `rate(_sum) / rate(_count)`.
2. **Queue Time quantiles are fiction at low load.** vLLM's smallest queue-time bucket is 0.3 s. All 362 requests queued under it (mean 7.8 µs), so `histogram_quantile` interpolated inside that one bucket and drew p50 = 150 ms, p99 = 297 ms on an idle server. The panel now shows the mean only.
3. **Max Generation Token in Sequence Group** only differs from generation length under parallel sampling (`n>1`), and its `rate(_sum)` has no meaning. Replaced with the missing **Request Throughput** (req/s) panel.

The full panel-to-definition table is in the observability README.

## Runs

### Trickle (2 workers, 2 min)

61/61 OK. Server-side mean TTFT 82 ms, mean ITL 36.6 ms. curl saw 3.2-4.2 s per request against about 2.6 s of server time: each curl opens a new TLS connection to us-east. Details: `artifacts/week3-trickle.md`.

### Ramp (llm-slo-bench, 6 min)

Linear ramp 0 -> 0.5 -> 1 -> 2 req/s over three 2-minute stages, `max_in_flight: 8`, 20-token prompt ending in `/no_think`, `max_completion_tokens: 128`.

- 300/300 succeeded, 0 errors, 0 dropped. Both gates passed (exit 0).
- 72 output tokens per request on average; all stopped on EOS.
- Peak 5 requests running, 0 waiting, KV cache under 1%. The server was nowhere near saturation; this is the low-load baseline, not capacity.

| Metric (ms) | p50 | p95 | p99 | mean |
|---|---|---|---|---|
| Client TTFB (first byte, role-only chunk) | 98.9 | 126.8 | 155.1 | 104.3 |
| Client TTFT (first content token) | 194.0 | 225.5 | 247.8 | 195.3 |
| Server TTFT (vLLM histogram) | - | - | - | 94.5 |
| Client chunk ITL | 36.8 | 39.9 | 49.1 | 36.6 |
| Client request duration | 2791 | 2968 | 3090 | 2799 |

Per-request generation speed: 27.7 tokens/s mean.

## Findings

1. **Client TTFT is the server's TTFT plus about 100 ms of network.** Client mean 195 ms, server mean 94.5 ms. The 100 ms is the round trip and TLS from the Pi to Modal us-east. Both are called "TTFT"; only one is what a user feels.
2. **The network doesn't touch inter-token latency.** Client chunk ITL (36.6 ms mean) matches the server's (about 37 ms on the dashboard during the ramp, 36.6 ms in the trickle run): once the stream is open, tokens arrive at the GPU's pace.
3. **Decode runs at about 73% of the A10's memory bandwidth.** Reading 16 GB of BF16 weights at 600 GB/s takes at least 27 ms per token; observed 36.6 ms. This is week 1's roofline on my own server, in eager mode.
4. **A dashboard percentile is only as good as its buckets.** The queue-time p99 of 297 ms was invented by interpolation. Server-side TTFT p99 lands between the 0.1 s and 0.25 s buckets, so it can't be compared with the client's HDR-histogram p99, even though both happened to read about 247 ms.
5. **Little's law checks out.** 2 req/s x 2.8 s per request = 5.6 in flight; the Scheduler panel peaked at 5 running.
6. **Qwen3 `/no_think` still emits `<think></think>`** without a reasoning parser. The first content token is `<think>`, so semantic TTFT is honest (it's the first generated token) but includes 3 tokens the user never needs.

## What Surprised Me

- How much of the example dashboard was wrong: three of fourteen panels had no per-request meaning, and the percentiles were confidently wrong at low load.
- That the end-to-end client time was dominated by connection setup in the trickle runs, not by the model.

## Bonus: Same Ramp Through My Gateway

Ran the identical ramp through [sre-inference-gateway](https://github.com/crypticseeds/sre-inference-gateway) in front of the same vLLM server. Full write-up lives in that repo (`docs/measurements/2026-10-18-vllm-modal/`); llm-slo-bench's README keeps only the direct run, since it's a different workload.

What I learned:

- **The first real-backend request found a bug the mocks never could:** the gateway forwarded `"name": null`, which vLLM rejects. Mocks accept anything.
- **The proxy cost is all in getting started.** The gateway added about 100 ms to the first token and nothing between tokens.
- **Measure before blaming the language.** The gateway's own timer showed its Python code cost about 3 ms; the 100 ms was on its connection to Modal. The likely cause is connection reuse (Go keeps connections open forever, httpx drops them after 5 s idle), not Python speed. Not yet proven.

## Next

Week 4: SGLang and batching. The capacity question (where `num_requests_waiting` lifts off zero) is week 5.
