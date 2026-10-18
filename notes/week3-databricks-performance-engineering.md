# Week 3 Thu (part 1): Databricks LLM Inference Performance Engineering

Source: [LLM Inference Performance Engineering: Best Practices](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices), MosaicML and Databricks, October 2023.

You already have a full summary from week 1 in [llm-inference-performance-engineering.md](llm-inference-performance-engineering.md). This file is the week 3 view: only what feeds the dashboard and the "map every metric to a panel" task.

## Main Idea

Pick the goal first (interactive latency, throughput, or cost), then measure the metrics that describe it, and keep them separate. One headline number hides the tradeoffs.

## Metric Definitions It Gives You

| Metric | Definition | Driven by |
|---|---|---|
| Time to First Token (TTFT) | Request submitted to first output token. Includes queueing and prefill | Prefill (compute-bound), queue depth |
| Time Per Output Token (TPOT) | Average time between output tokens for one request. `1 / TPOT` = tokens/s per user | Decode (memory-bandwidth-bound) |
| End-to-end latency | `TTFT + TPOT x (output_tokens - 1)` | Mostly output length |
| Throughput | Output tokens/s or requests/s across all users | Batch size |
| Model Bandwidth Utilization (MBU) | `(model bytes + KV bytes read per step) / TPOT / peak bandwidth` | How close decode is to the hardware limit |

## What It Adds to Measurement

- **Throughput and latency trade against each other.** Larger batches raise total throughput and per-request latency together. The right operating point is the knee of the throughput-latency curve, not the largest batch that fits.
- **Output length dominates e2e latency**, because output tokens are sequential. Input length mostly shows up as TTFT and KV cache memory.
- **MBU is a derived panel, not a vLLM metric.** You can compute it from TPOT, model size and the GPU spec sheet. It tells you whether decode still has software headroom.
- **Tokenizers differ.** Tokens/s across model families is not apples to apples.

## Benchmarking Rules From the Post

1. Define the workload (input and output lengths) before a latency target.
2. Report TTFT and TPOT separately, plus throughput.
3. Sweep load to expose the saturation curve.
4. Record hardware, engine version, precision, batch size.
5. Measure end to end; spec-sheet limits ignore software and communication overhead.

## Key Points to Retain

- TTFT = prefill + queue. TPOT = decode. Different bottlenecks, different panels.
- Throughput up usually means per-user latency up.
- MBU tells you how much faster decode could get on this GPU.
- The post's absolute numbers are 2023 hardware; keep the method, not the figures.
