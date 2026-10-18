# Week 1 Revision Guide: The Mental Model

## Summary

Before optimizing a workload, identify whether it is limited by computation, memory movement, or overhead. Apply an optimization that reduces that specific limit.

## 1. The Three Performance Limits

### Computation

The Graphics Processing Unit (GPU) is busy performing mathematical work. Performance is limited by its peak **Floating-Point Operations Per Second (FLOPS)**. Large matrix multiplications can enter this regime.

### Memory movement

The GPU waits for weights or activations to move from memory to its compute units. Performance is limited by memory bandwidth, measured in bytes per second.

### Overhead

The GPU waits for work to be prepared or launched. Common causes include Python execution, framework dispatch, kernel launches, and synchronization.

**Key rule:** more compute power does not fix a memory-bound task. More memory bandwidth does not fix a compute-bound task.

## 2. Graphics Processing Unit Basics

A Central Processing Unit (CPU) has a small number of powerful cores suited to varied, sequential work. A GPU has many simpler execution units suited to performing similar operations across large amounts of data.

- **Threads** are individual lanes of work. Hardware schedules them in groups, commonly called warps.
- **Streaming Multiprocessors (SMs)** execute groups of threads and contain compute units, registers, and shared memory.
- **Single Instruction, Multiple Threads (SIMT)** means many threads execute the same instruction on different data. Different branches between threads reduce efficiency.
- **Tensor Cores** are specialized units for matrix multiplication.

### Memory hierarchy

Memory close to computation is fast but small. Memory farther away is larger but slower.

- Registers and Static Random-Access Memory (SRAM), including shared memory and caches, are close to the compute units.
- High Bandwidth Memory (HBM), a form of Dynamic Random-Access Memory (DRAM), stores model weights and large tensors.
- Every unnecessary trip to HBM costs time.

Think of compute units as a factory, HBM as a warehouse, and memory bandwidth as the road between them. A larger factory is wasted if the road cannot deliver materials quickly enough.

## 3. Arithmetic Intensity and the Roofline Model

**Arithmetic intensity** measures how much computation is performed for each byte moved from memory.

```text
Arithmetic intensity = Floating-Point Operations (FLOPs) / bytes moved
```

A **Floating-Point Operation (FLOP)** counts work. **Floating-Point Operations Per Second (FLOPS)** measures a rate.

The **roofline model** estimates the maximum performance allowed by computation and memory bandwidth:

```text
Attainable FLOPS = min(peak FLOPS, memory bandwidth x arithmetic intensity)
```

- Low arithmetic intensity usually means the operation is **memory-bound**.
- High arithmetic intensity usually means the operation is **compute-bound**.
- The **ridge point** is where both limits meet.

```text
Ridge point = peak FLOPS / peak memory bandwidth
```

An operation below the ridge point needs more data reuse or fewer bytes moved. An operation above it benefits from faster computation or fewer FLOPs.

The roofline is an upper bound, not an exact speed prediction. Real performance also includes overhead, communication, synchronization, and imperfect hardware utilization.

## 4. Prefill and Decode

Large Language Model (LLM) inference has two different phases.

### Prefill

Prefill processes the prompt tokens largely in parallel. Large matrix operations reuse model weights across many tokens, producing high arithmetic intensity. Long prefills can therefore become compute-bound.

Prefill strongly affects **Time to First Token (TTFT)**.

### Decode

Decode generates one token at a time because each new token depends on the previous token. At a small batch size, every step reads most of the model weights but performs little work with each byte. Decode is therefore usually memory-bound.

Decode strongly affects **Time Per Output Token (TPOT)**, also called inter-token latency.

### Key-Value cache

The **Key-Value (KV) cache** stores attention keys and values for earlier tokens. It prevents the model from recomputing old attention state during every decode step.

The KV cache saves computation but consumes memory. Its size grows with the number of requests and their sequence lengths.

### Batching

Batching lets one model-weight read support several token computations. This increases arithmetic intensity and total throughput.

The tradeoffs are:

- Individual users may wait longer.
- A larger KV cache consumes more memory.
- After the workload becomes compute-bound, increasing the batch size may add latency without improving throughput.

## 5. Matching Optimizations to Bottlenecks

| Bottleneck | Useful responses | Why they help |
|---|---|---|
| Memory bandwidth | Operator fusion, quantization, batching, data reuse | Move fewer bytes or perform more work per byte |
| Computation | Faster matrix kernels, lower-precision computation, fewer FLOPs, more compute hardware | Reduce or accelerate mathematical work |
| Overhead | Combine small operations, compile graphs, reduce kernel launches | Keep the GPU busy |
| Memory capacity | Quantization, KV cache management, model parallelism | Make weights and request state fit in memory |

### Operator fusion

A GPU kernel is a unit of work launched on the GPU. Separate kernels may write an intermediate tensor to HBM and immediately read it again.

**Operator fusion** combines compatible operations into one kernel so intermediate values can remain in faster on-chip memory. It reduces memory traffic and kernel-launch overhead.

In a memory-bound operation, adding some mathematical work can be nearly free. Recomputing a small value can even be faster than loading it from memory.

## 6. Worked 7-Billion-Parameter Example

Assume:

- 7 billion parameters
- 16-bit weights, which use 2 bytes each
- Batch size one

### Weight size

```text
7 billion parameters x 2 bytes = about 14 gigabytes
```

### Work for one decoded token

A useful approximation is 2 FLOPs per parameter:

```text
2 x 7 billion parameters = about 14 billion FLOPs
```

### Arithmetic intensity

```text
14 billion FLOPs / 14 billion bytes = about 1 FLOP per byte
```

For batch-size-one decode:

```text
Approximate arithmetic intensity = 2 / bytes per weight
```

- 16-bit weights: about 1 FLOP per byte
- 8-bit weights: about 2 FLOPs per byte
- 4-bit weights: about 4 FLOPs per byte

Modern GPUs usually have much higher ridge points, so batch-size-one decode is strongly memory-bound.

This calculation is intentionally approximate. It focuses on model-weight reads and the main matrix operations.

## 7. Personal Calculation Worksheet

```text
Model: ____________________
Parameter count: ____________________
Bytes per weight: ____________________

Decode FLOPs per token ~= 2 x parameters = ____________________
Weight bytes ~= parameters x bytes per weight = ____________________
Arithmetic intensity ~= FLOPs per token / weight bytes = ____________________

GPU peak FLOPS: ____________________
GPU memory bandwidth: ____________________
GPU ridge point = peak FLOPS / memory bandwidth = ____________________

Conclusion: arithmetic intensity is [below / above] the ridge point,
so decode is likely [memory-bound / compute-bound].
```

## 8. Revision Questions

1. What are the three performance limits?
2. What does arithmetic intensity measure?
3. Why is low-batch decode usually memory-bound?
4. Why is prefill more likely to be compute-bound?
5. Why does operator fusion improve performance?

### Answers

1. Computation, memory bandwidth, and overhead.
2. The number of Floating-Point Operations (FLOPs) performed for each byte moved from memory.
3. Each sequential token step reads most model weights but performs little work with each byte.
4. Prompt tokens are processed in parallel, allowing weights to be reused in large matrix operations.
5. It avoids intermediate reads and writes to High Bandwidth Memory (HBM) and reduces kernel-launch overhead.

## Key Points to Retain

- Classify the bottleneck before choosing an optimization.
- Data movement can cost more time than the operation performed on the data.
- Arithmetic intensity connects software work to hardware limits.
- Prefill and decode are different workloads and should be measured separately.
- Real measurements determine whether the simplified model is correct for a service.

## Further Reading

- [Making Deep Learning Go Brrrr From First Principles](https://horace.io/brrr_intro.html) - performance regimes, operator fusion, and overhead.
- [Transformer Inference Arithmetic](https://kipp.ly/transformer-inference-arithmetic/) - model weights, Key-Value (KV) caching, batching, and latency estimates.
- [Large Language Model Inference Performance Engineering](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices) - prefill, decode, serving metrics, and Model Bandwidth Utilization (MBU).
