# Week 5 Fri: Practical Strategies for vLLM Performance Tuning

Source: [Practical strategies for vLLM performance tuning](https://developers.redhat.com/articles/2026/03/03/practical-strategies-vllm-performance-tuning) (Red Hat Developer, 2026-03-03).

## Main Idea

There is no single magic flag. Tuning is a loop: **benchmark a representative workload, change one knob, benchmark again**, and retune whenever the request shapes or the model change. The article covers how to build the benchmark and four knobs that matter most.

## 1. Start With a Representative Workload

- Use a tool like GuideLLM for repeatable runs. Avoid purely synthetic or oversimplified benchmarks.
- **Input and output shapes** (token counts, variability, response size) drive KV cache use and scheduling.
- **Repeated text** (reused prompts, tool context) changes results, because vLLM caches recurring prefixes.
- Capture throughput, TTFT and end-to-end latency, and set SLOs at **p95 and p99**.

## 2. The Four Knobs

### GPUs per replica

Example: two nodes with 8 H100s each. Two 8-GPU replicas, four 4-GPU replicas, eight 2-GPU replicas, and so on.

- Smaller replicas: more redundancy, easier to schedule, but less memory each for KV cache.
- Larger replicas: more KV cache per replica, more parallelism overhead.
- **Method:** find the fewest GPUs that load the model *with enough KV cache*. Deploy as many replicas of that size as fit. Test across concurrencies. Then trade replicas for GPUs per replica and compare.

### `--gpu-memory-utilization`

- Share of GPU memory for weights plus KV cache. Default **0.9**; the rest covers CUDA graphs and runtime overhead.
- The unused 10% can reach **8 GB on an H100**. Reclaiming it means more KV cache, more concurrent requests.
- Too high crashes the pod.
- **Method:** raise until startup fails or the server is unstable under heavy load, then back off for a margin.

### `--kv-cache-dtype`

- Default is the model's dtype. Lower precision such as **`fp8`** halves memory per token versus BF16, so more concurrent requests fit.
- It can cost output quality.
- **Method:** use the lowest precision the hardware and model support that keeps quality acceptable, check with automated evals, roll back on regressions.

### `--max-num-seqs`

- Maximum requests processed at once; the rest wait in the queue.
- Protects throughput near its best under heavy load, at the cost of higher TTFT and end-to-end latency for queued requests.
- **Method:** measure throughput, TTFT and latency across concurrencies. Find where **throughput plateaus while latency keeps rising**. Start there, then move up for throughput stability or down for latency balance.

## 3. How This Maps to My Runs

| Knob | My setting | What the runs showed |
|---|---|---|
| `--max-num-seqs` | 16 (A10), 256 (H100) | The A10 saturated at 16 running; the H100 filled 256 from 500 concurrent, exactly where throughput stopped growing (327 req/s). That is the plateau the article says to look for |
| `--gpu-memory-utilization` | 0.90 | KV cache peaked under 6% with 88-token requests, so more memory would not have helped this workload |
| `--kv-cache-dtype` | BF16 (default) | Same: memory was never the limit. FP8 KV matters for long contexts, which week 6 and 7 will test |
| GPUs per replica | 1 | An 8B model fits one GPU; replicas vs tensor parallel only matters at larger models or for HA |

## Key Points to Retain

- Iterate: one change, one benchmark, compare.
- Fewest GPUs that hold the model and enough KV cache, then as many replicas as fit.
- `--gpu-memory-utilization`: push until it breaks, back off.
- FP8 KV cache: more concurrency, verify quality.
- `--max-num-seqs`: set it at the knee where throughput plateaus and latency climbs.
- Which knob matters depends on the workload. With short requests, the limit is scheduling, not memory.
