# Week 3 Fri: Modal, How to Benchmark LLM Engines

Source: [How to Benchmark LLM Engines](https://modal.com/llm-almanac/how-to-benchmark), Modal LLM Engineer's Almanac. Methodology built on Red Hat's `guidellm`, implemented in Modal's open-source `stopwatch` framework.

## Main Idea

A single benchmark number at one load level is meaningless. Instead:

1. **Bound** the engine: find its maximum throughput and its minimum latency.
2. **Sweep** constant request rates between those bounds.
3. Read latency at each rate and pick the highest rate that still meets your SLO.

## 0. Prerequisites Before Benchmarking

- The model, plus which quantization levels are acceptable.
- Typical input and output token counts for your workload.
- The engine (vLLM, SGLang, ...).
- A latency objective: TTFT if streaming, Time to Last Token (TTLT) if not.

## 1. Upper Bound: Maximum Throughput (`guidellm` throughput mode)

- Send a whole batch of requests at once (an "infinite rate for an infinitely short time"), wait for all to finish, divide requests by duration.
- Exposes maximum parallelism, so the engine reaches peak aggregate throughput at the cost of latency.
- **The latency numbers from this run are discarded.** Nobody runs a service that way, so they say nothing about user experience.
- Queueing theory: if arrival rate stays above this maximum, the queue and latency grow without bound until the system breaks.
- Reported in **requests per second, not tokens per second**, because tokens are not fungible across workloads.

## 2. Lower Bound: Minimum Latency (`guidellm` synchronous mode)

- One request at a time; send the next only after the previous returns.
- No contention, so this is the best latency the engine can give.
- Throughput collapses. It is a very expensive way to run a GPU, only justified by a tight SLO.
- Its rate also gives the **minimum sensible request rate**: the lowest rate at which the engine is always busy.

## 3. The Sweep (`guidellm` constant mode)

- Send requests at fixed rates between the min and max, **without waiting** for responses.
- This resembles real traffic: aggregate client demand changes slowly relative to the request rate.
- Collect latency percentiles at each rate. This curve is the actual result.

```text
latency
  ^                                   x  <- saturation: queue grows, discard
  |                               x
  |                         x
  |              x    x
  |  x     x                              SLO line --------
  +------------------------------------------> request rate
  sync                                   throughput-mode max
```

## 4. Latency Metrics, From the Client's View

| Metric | When it matters | Notes |
|---|---|---|
| Time to First Token (TTFT) | Streaming to humans | Measures prefill. Observed range ~200 ms to tens of seconds |
| Time to Last Token (TTLT) | Non-streaming, machine clients (e.g. parsing JSON) | Controlled by the model, since it decides when to emit the stop token |
| Inter-Token Latency (ITL) / TPOT | Engineering metric, not a client SLO | ~1 ms (small model, big GPU) to a large fraction of a second |

To keep TTLT measurable, benchmarks keep generating past stop tokens (unrealistic but clean).

## 5. Estimating Latency Without Running Production

- **Speed-of-light bound:** every decode step must stream all weights from GPU memory, so `ITL >= model bytes / memory bandwidth`. For an 8B model at 1 byte per parameter on an H100 (~4 TB/s per the article): `8 GB / 4 TB/s = 2 ms`. (The article says "half a millisecond", which does not match its own numbers; redo the arithmetic yourself.) They observed ~5 ms in practice when optimizing for latency.
- Spec-sheet bandwidth gives a lower bound on the **mean**. SLOs care about **percentiles**, which have no clean formula, hence benchmarking.
- **"Three nineties" rule of thumb** for estimating tail TTLT across varying output lengths:

```text
p90 TTLT ~= p90 TTFT + p90 ITL x output_tokens
```

- ITL rises with input length and with tokens already generated.
- Match your data to production at least roughly. Natural language (they used *Pride and Prejudice*) behaves differently from code, where speculative decoding succeeds more often.

## Key Points to Retain

- Benchmark is a curve, not a point: bound, then sweep request rate.
- Throughput mode gives max RPS; its latency is thrown away.
- Synchronous mode gives the latency floor and the minimum busy rate.
- Constant-rate sweep is the realistic measurement.
- Past saturation, latency is pure queueing; discard those points.
- Report RPS for a defined workload rather than raw tokens/s.
- `p90 TTFT + p90 ITL x tokens` estimates p90 TTLT.
