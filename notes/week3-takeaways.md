# Week 3 Takeaways: Measurement

Quick recap of the week 3 readings. Full notes:

- [vLLM metrics design](week3-vllm-metrics-design.md)
- [Databricks performance engineering](week3-databricks-performance-engineering.md)
- [NVIDIA benchmarking fundamentals](week3-nvidia-benchmarking-fundamentals.md)
- [Modal: how to benchmark](week3-modal-how-to-benchmark.md)

## The One-Line Version

**A benchmark is a latency-vs-load curve for a defined workload, not a single number, and every metric on it needs a stated definition.**

## Core Metrics Cheat Sheet

| Metric | Formula / meaning | Phase | vLLM metric |
|---|---|---|---|
| TTFT | Arrival to first token. Includes tokenization, queue, prefill (+ network client-side) | Prefill | `vllm:time_to_first_token_seconds` |
| ITL | Gap between consecutive outputs | Decode | `vllm:inter_token_latency_seconds` |
| TPOT | `(e2e - TTFT) / (out_tokens - 1)`, per request | Decode | `vllm:request_time_per_output_token_seconds` |
| E2E / TTLT | `TTFT + TPOT x (out_tokens - 1)` | Both | `vllm:e2e_request_latency_seconds` |
| Queue time | Arrival to first scheduled | Scheduler | `vllm:request_queue_time_seconds` |
| System throughput | Output tokens/s or requests/s, all users | All | `rate(vllm:generation_tokens_total[1m])` |
| Per-user speed | `OSL / e2e`, approaches `1 / ITL` | Decode | derived |
| MBU | `bytes read per step / TPOT / peak bandwidth` | Decode | derived from TPOT + spec sheet |

## Ten Things to Remember

1. **Prefill drives TTFT (compute-bound); decode drives ITL/TPOT (memory-bandwidth-bound).** Keep them on separate panels.
2. **Server-level gauges explain request-level histograms.** TTFT spikes? Check `num_requests_waiting` and `kv_cache_usage_perc`.
3. **`num_requests_waiting` sustained above 0 = saturated.** This is the autoscaling signal (week 8), not CPU.
4. **`kv_cache_usage_perc` near 1 = preemptions,** which restart prefill and inflate latency. Do not benchmark at that point.
5. **Throughput and per-user latency trade off.** More concurrency raises system tokens/s until saturation, while each user gets slower.
6. **Same name, different formula.** GenAI-Perf ITL excludes TTFT, LLMPerf includes it. vLLM's Prometheus TPOT counts one-token requests as 0, `vllm bench serve` drops them. Always state the tool.
7. **Bound, then sweep.** Max throughput (all at once) and min latency (one at a time) bracket the range; constant-rate sweep in between is the real result. Throughput-mode latency and post-saturation points get discarded.
8. **Concurrency above max batch size measures the queue, not the engine.** Sweep from 1 to just past max batch.
9. **Fix the workload.** Report exact ISL/OSL (use realistic distributions), set `ignore_eos=True`, fix sampling, pin versions. Tokens differ across tokenizers, so prefer requests/s for a defined workload.
10. **Quick tail estimate:** `p90 TTLT ~= p90 TTFT + p90 ITL x output_tokens`. Spec-sheet bandwidth (`model bytes / bandwidth`) only bounds the mean ITL.

## PromQL You Will Actually Use

```promql
# p99 TTFT over 5 minutes
histogram_quantile(0.99, sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[5m])))

# p50 inter-token latency
histogram_quantile(0.50, sum by (le) (rate(vllm:inter_token_latency_seconds_bucket[5m])))

# Output tokens per second
rate(vllm:generation_tokens_total[1m])

# Prefix cache hit rate
rate(vllm:prefix_cache_hits_total[5m]) / rate(vllm:prefix_cache_queries_total[5m])

# Queue depth and KV pressure (plot raw)
vllm:num_requests_waiting
vllm:kv_cache_usage_perc
```

Check the exact names against your server's `curl localhost:8000/metrics`; counter suffixes (`_total`) vary by version.

## Week 3 Deliverable Checklist

A live Grafana dashboard with, at minimum:

- [ ] TTFT p50 / p95 / p99
- [ ] ITL (or TPOT) p50 / p95 / p99
- [ ] Output tokens/s and requests/s
- [ ] Queue depth (`num_requests_waiting`) next to `num_requests_running`
- [ ] KV cache usage
- [ ] Every panel traceable to a definition above; delete the rest

## Sources Note

- The vLLM doc moved to `docs.vllm.ai/en/latest/design/metrics/`.
- The NVIDIA post moved to `developer.nvidia.com/blog/llm-benchmarking-fundamental-concepts/`. The README links are stale.
