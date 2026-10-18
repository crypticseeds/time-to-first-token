# Week 3 Mon: vLLM Metrics Design

Source: [vLLM Metrics design doc](https://docs.vllm.ai/en/latest/design/metrics/) (the README link `design/v1/metrics/` now redirects here).

## Main Idea

vLLM exposes two kinds of metrics on a Prometheus `/metrics` endpoint, every name prefixed `vllm:` and labelled with `model_name`:

1. **Server-level metrics** (Gauges and Counters): the state of the engine right now, such as how many requests are running or how full the Key-Value (KV) cache is.
2. **Request-level metrics** (Histograms): the size and timing of individual requests, such as Time to First Token (TTFT). These are the Service-Level Objectives (SLOs) you alert on.

The mental model: **server-level metrics explain request-level metrics.** If TTFT p99 jumps, look at `num_requests_waiting` and `kv_cache_usage_perc` to see why.

## 1. Prometheus Metric Types

| Type | Behaviour | Example | How to graph it |
|---|---|---|---|
| Counter | Only goes up, resets on restart | `vllm:generation_tokens_total` | `rate(...[1m])` to get per-second |
| Gauge | Goes up and down | `vllm:num_requests_running` | Plot the raw value |
| Histogram | Counts samples into buckets | `vllm:time_to_first_token_seconds` | `histogram_quantile(0.99, rate(..._bucket[5m]))` |

## 2. The Metrics That Matter

### Server-level (gauges and counters)

| Metric | What it actually measures |
|---|---|
| `vllm:num_requests_running` | Requests in the current model execution batch |
| `vllm:num_requests_waiting` | Requests queued, not yet scheduled. Rising = saturation |
| `vllm:kv_cache_usage_perc` | Fraction of KV cache blocks in use (0 to 1). Near 1 = preemptions coming |
| `vllm:prompt_tokens_total` | Prompt tokens processed (prefill work) |
| `vllm:generation_tokens_total` | Output tokens generated (decode work) |
| `vllm:prefix_cache_queries` / `_hits` | Prefix cache lookups and hits. Divide the rates for hit rate |
| `vllm:request_success_total` | Finished requests, labelled by `finished_reason` (`stop`, `length`, `abort`) |

### Request-level (histograms)

| Metric | Interval |
|---|---|
| `vllm:time_to_first_token_seconds` | Frontend arrival (tokenization start) to first token |
| `vllm:inter_token_latency_seconds` | Gap between consecutive streamed outputs, one sample per output |
| `vllm:request_time_per_output_token_seconds` | Per request: `(e2e - TTFT) / (output_tokens - 1)` |
| `vllm:e2e_request_latency_seconds` | Frontend arrival to final token |
| `vllm:request_queue_time_seconds` | Time waiting before first scheduled |
| `vllm:request_prefill_time_seconds` | Scheduled to first token |
| `vllm:request_decode_time_seconds` | First token to last token |
| `vllm:request_prompt_tokens` / `_generation_tokens` | Input and output length distributions |

### ITL vs TPOT: not the same metric

- **Inter-Token Latency (ITL)** is recorded once per streamed output event.
- **Time Per Output Token (TPOT)** is recorded once per finished request.
- They diverge when one output bundles several tokens (speculative decoding) or when averaged over different weights (many tokens vs many requests).
- Requests with one or fewer output tokens get TPOT = 0 in Prometheus, but `vllm bench serve` excludes them. So the dashboard and the benchmark tool can disagree on TPOT.

## 3. How the Timings Are Collected

V1 keeps metrics work out of the hot loop:

- **EngineCore** (inner loop, performance-critical) only records timestamps and scheduler stats.
- **AsyncLLM / API server** (outer loop, overlapped with GPU work) turns those into Prometheus observations.

The engine core stamps four events per request:

```text
QUEUED ----> SCHEDULED ----> NEW_TOKENS (first) ----> ... ----> NEW_TOKENS (last)
  |  queue interval  |  prefill interval  |        decode interval        |
                     |<------------- inference interval ------------------>|
```

- `PREEMPTED` sends a request back to the waiting queue; it restarts prefill when rescheduled. A decode-time preemption inflates ITL, decode and inference intervals.
- Intervals use **monotonic time**, and only timestamps from the **same process** are compared, because each process's monotonic clock has its own zero. That is why the engine core, not the frontend, must stamp QUEUED and SCHEDULED.
- TTFT and e2e latency are measured from the **frontend** `arrival_time`, so they include tokenization.

## 4. Design Decisions Worth Remembering

- **Expose counters, not ratios.** Prefix cache hit rate is exposed as `queries` and `hits` counters so you compute the rate over any window you want: `rate(hits[5m]) / rate(queries[5m])`. The doc suggests the same for speculative decoding acceptance.
- **Info-style metrics.** `vllm:cache_config_info` is a gauge fixed at 1 whose labels carry config (`block_size`, `gpu_memory_utilization`, `enable_prefix_caching`). Useful for comparing instances.
- **Legacy swap metrics are gone in V1.** `num_requests_swapped` and `cpu_cache_usage_perc` belonged to CPU swapping; V1 preempts and recomputes instead, relying on prefix caching. Grafana panels that still query them will show nothing.
- **KV block residency histograms** (sampled via `--kv-cache-metrics-sample`): block lifetime, idle time before eviction, and reuse gap. Good for spotting stranded cache.
- **Python process metrics disappear** when `--api-server-count > 1` (multiprocess mode).
- **Metrics have a cost.** They are hard to deprecate, can slow the server, and add maintenance. Higher-resolution timings (`model_forward_time`, `model_execute_time`) only appear with OpenTelemetry tracing and `--collect-detailed-traces`.

## 5. Autoscaling: the Open Question

The doc names the goal but not the answer: expose what is needed to detect the **saturation point**, where raising the request rate stops increasing throughput and only adds latency. Queue depth (`num_requests_waiting`) is the practical signal, which is why week 8 autoscales on queue depth, not CPU.

## Key Points to Retain

- Server-level gauges explain request-level histograms.
- `num_requests_waiting` > 0 sustained means you are past capacity.
- `kv_cache_usage_perc` near 1 means preemptions and recomputed prefills.
- ITL is per output event, TPOT is per request. Know which one your panel shows.
- TTFT includes tokenization and queueing, not just prefill.
- Compute rates and ratios in PromQL from counters, never trust a pre-baked ratio gauge.
