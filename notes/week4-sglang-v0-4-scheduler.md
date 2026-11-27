# Week 4 Buffer: SGLang v0.4 Zero-Overhead Scheduler

Source: [SGLang v0.4: Zero-Overhead Batch Scheduler, Cache-Aware Load Balancer, Faster Structured Outputs](https://lmsys.org/blog/2024-12-04-sglang-v0-4/) (LMSYS, 2024-12-04).

## Main Idea

Once the GPU kernels are fast, the **CPU becomes the bottleneck**. The article says unoptimized engines can spend up to **half their time** on CPU work: batch scheduling, memory allocation, prefix matching. While the CPU prepares the next batch, the GPU sits idle.

v0.4 hides that work by running the scheduler **one batch ahead** of the GPU. It also adds a cache-aware router for multi-replica serving, data-parallel attention for DeepSeek, and a faster grammar backend.

## 1. Zero-Overhead Batch Scheduler

```text
before:  CPU [schedule 1]          [schedule 2]          [schedule 3]
         GPU            [batch 1]             [batch 2]             ...   gaps

after:   CPU [schedule 1][schedule 2][schedule 3]
         GPU             [batch 1  ][batch 2  ][batch 3]                  no gaps
```

- Idea from NanoFlow: overlap CPU scheduling with GPU compute.
- The scheduler prepares all metadata for batch N+1 (including radix cache operations) while batch N runs.
- Dependencies on tokens not generated yet are handled with "future tokens" and CUDA events.
- Nsight profile: five consecutive decode batches with **no GPU idle time**.
- **1.1x** faster than the previous version, **1.3x** faster than other baselines. Largest gains on **small models** and **large tensor parallel** sizes, where GPU steps are shortest and CPU overhead is the biggest share.
- On by default; `--disable-overlap` turns it off.
- Benchmark: Llama-3.2-3B-Instruct, `sglang.bench_serving` with 500 random prompts, 4,096 input and 2,048 output tokens.

## 2. Cache-Aware Load Balancer

- A router in front of several workers predicts each worker's prefix hit rate and sends the request to the best match.
- It keeps an **approximate radix tree** of each worker's cache, updated lazily, with no worker synchronization.
- Written in Rust, shipped as the separate `sglang-router` package.
- On 8x A100 80 GB with a shared-prefix workload: throughput **82,665 to 158,596 tokens/s** (up to 1.9x) and cache hit rate **20% to 75%** (3.8x) vs v0.3.

This is the data-parallel meta-tree from the [paper](week4-sglang-paper.md) turned into a product. Round-robin load balancing spreads each prefix across all replicas and destroys the hit rate.

## 3. Data-Parallel Attention for DeepSeek

DeepSeek's Multi-head Latent Attention (MLA) has a single KV head, so tensor parallelism duplicates the KV cache on every GPU. Running attention data-parallel instead (all-gather before the MoE layer) frees that memory for bigger batches: up to **1.9x** decode throughput on 8x H100. Flag: `--enable-dp-attention`, DeepSeek only.

## 4. Structured Output with xgrammar

`--grammar-backend xgrammar`: up to **10x** faster JSON decoding than other open-source options.

## Key Points to Retain

- At small model sizes and short steps, CPU scheduling overhead is a real share of latency. Overlapping it with GPU work is the fix.
- This is a different source of speedup from prefix reuse (RadixAttention) and batching policy (continuous batching, chunked prefill).
- Prefix caching across replicas needs cache-aware routing, or the hit rate collapses.
- Relevant to my H100 run: with 72-token prompts and 16-token outputs, GPU steps are tiny, so CPU and request-handling overhead matter more than usual. Most H100 time to first token above 250 concurrent was spent before the scheduler queue.
