# Week 5 Takeaways: Load Testing

Quick recap of the week 5 readings. Full notes:

- [GuideLLM introduction](week5-guidellm-intro.md) (Mon)
- [GuideLLM repository](week5-guidellm-repo.md) (Tue)
- [`vllm bench serve`](week5-vllm-bench-serve.md) (Wed)
- [perf_analyzer and GenAI-Perf](week5-perf-analyzer-genai-perf.md) (Thu)
- [vLLM tuning strategies](week5-vllm-tuning-strategies.md) (Fri)

What I built and ran: [Week 5 build log](week5-build-log.md).

## The One-Line Version

**Capacity is the highest load that still meets a stated SLO, on a pinned workload, from a client that actually delivered that load.**

## Load Shapes Cheat Sheet

| Shape | GuideLLM | `vllm bench serve` | perf_analyzer | Answers |
|---|---|---|---|---|
| One at a time | `synchronous` | `--max-concurrency 1` | concurrency 1 | Best-case latency |
| Fixed arrival rate (open loop) | `constant`, `poisson` | `--request-rate N` (`--burstiness`) | request rate | Where does the queue start growing? |
| Fixed in-flight count (closed loop) | `concurrent` | `--request-rate inf --max-concurrency N` | concurrency | Throughput vs in-flight requests |
| Everything at once | `throughput` | `--request-rate inf` | - | Maximum throughput |
| Full curve | `sweep` | (script it) | - | All of the above |

Open loop keeps sending when the server falls behind, so it shows saturation. Closed loop can never have more than N in flight, so it shows how throughput scales with concurrency.

## Ten Things to Remember

1. **A sweep, not a point.** Synchronous baseline, maximum throughput, rates in between. The curve is the result.
2. **SLO first, then read the curve.** Metric, percentile, threshold (mine: p95 TTFT under 500 ms). Capacity per server x replicas = fleet size, for that workload only.
3. **Fix the lengths and report the formatted ones.** 64 configured prompt tokens became 72 after the chat template. Lengths set KV per request, which sets the curve.
4. **Check the client delivered the load.** GuideLLM's root dispatch delay above 0.2 s means the stage ran late. At my A10's 32 req/s it was 19 to 26 s: that stage measured saturation.
5. **Two tools agreeing on the verdict is the cross-check,** not identical milliseconds. Client location and drain vs time-window stopping move the absolute numbers.
6. **Same metric name, different definitions.** GuideLLM ITL excludes the first token; `vllm bench serve` TPOT drops one-token requests, Prometheus counts them as 0.
7. **`--max-num-seqs` sets the knee.** Throughput plateaus once the batch is full; beyond that, queued requests only add TTFT. My H100 vLLM run plateaued at 327 req/s exactly when 256 running filled.
8. **Push `--gpu-memory-utilization` until it breaks, then back off.** FP8 KV cache buys concurrency at a quality risk. Neither mattered for short requests.
9. **Watch where the time goes, not just how much.** On the A10 the scheduler queue explained TTFT. On the H100 past 250 concurrent it did not: most requests were waiting before the scheduler.
10. **Tools go stale.** GenAI-Perf is deprecated for AIPerf. Check the README before planning a session around a tool.

## PromQL I Actually Used

```promql
# Mean server TTFT over a rolling minute (reliable when buckets are coarse)
rate(vllm:time_to_first_token_seconds_sum[1m]) / rate(vllm:time_to_first_token_seconds_count[1m])

# Mean scheduler queue time
rate(vllm:request_queue_time_seconds_sum[1m]) / rate(vllm:request_queue_time_seconds_count[1m])

# Batch state, both engines
vllm:num_requests_running
vllm:num_requests_waiting
sglang:num_running_reqs
sglang:num_queue_reqs
```

## Week 5 Deliverable Checklist

From the roadmap: a scripted concurrency sweep past 1,000 concurrent, with p50, p95 and p99, cross-checked by two tools.

- [x] Scripted sweep to 1,000 concurrent on vLLM and SGLang (H100), zero request errors
- [x] p50 and p95 for TTFT, ITL and request latency in every run
- [ ] p99 with confidence: too few samples at low rates; reported only where the sample size supports it
- [x] Two-tool cross-check on the A10 rate sweep (GuideLLM and `vllm bench serve`)
- [ ] Two-tool cross-check on the H100 concurrency sweep (AIPerf would be the second tool)
- [x] Watched KV cache and queue for preemption: none at any stage
- [ ] Pre-publish benchmark checklist (Week 5 buffer)

## Sources Note

- The GuideLLM introduction uses the 2025 CLI (`guidellm --target ... --rate-type sweep`). Version 0.8.1 uses `guidellm run` with `--backend`, `--profile`, `--data` and `--constraint`.
- The perf_analyzer README marks GenAI-Perf deprecated and points to AIPerf.
