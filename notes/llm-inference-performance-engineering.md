# Large Language Model Inference Performance Engineering

Source: [LLM Inference Performance Engineering: Best Practices](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices), MosaicML and Databricks, October 2023.

## Main Idea

Large Language Model (LLM) inference is not controlled by one speed metric. A useful analysis separates:

- Prompt processing from token generation
- User latency from total system throughput
- Compute limits from memory-bandwidth limits
- Faster execution from lower model quality

Choose the target first: interactive responsiveness, maximum throughput, or minimum cost. Improving one can make another worse.

## 1. Two Inference Phases

```text
Input prompt -> Prefill -> First output token -> Decode -> Remaining output tokens
                mostly parallel                one token at a time
```

### Prefill

**Prefill** processes the input prompt. Prompt tokens can be handled largely in parallel, creating large matrix operations that reuse model weights. It is commonly compute-bound, especially for long prompts.

Prefill is the main contributor to **Time to First Token (TTFT)**.

### Decode

**Decode** generates output tokens autoregressively, meaning each new token depends on previous tokens. At a small batch size, every step reads most model weights but performs relatively little computation per byte moved.

Decode is therefore commonly memory-bandwidth-bound and is the main contributor to **Time Per Output Token (TPOT)**.

### Key-Value Cache

The **Key-Value (KV) cache** stores attention keys and values for earlier tokens. This avoids recomputing the same attention state during each decode step.

The tradeoff is memory usage. KV cache size grows with:

- Number of active requests
- Sequence length
- Number of model layers
- Number and size of KV attention heads
- Bytes used for each cached value

More KV cache capacity supports more concurrent or longer requests, but may require a larger Graphics Processing Unit (GPU) or more aggressive memory management.

## 2. Metrics That Must Stay Separate

### Time to First Token (TTFT)

Time from submitting a request until the first generated token arrives. It measures how quickly an interactive service begins responding.

TTFT includes queueing, prompt processing, and production of the first output token.

### Time Per Output Token (TPOT)

Average time between generated output tokens for one request. It measures how quickly the response appears after generation begins.

```text
Tokens per second per user ~= 1 / TPOT in seconds
```

For example, a TPOT of 0.1 seconds is approximately 10 tokens per second.

### End-to-End Latency

Total time required to complete the response.

```text
End-to-end latency ~= TTFT + TPOT x (output tokens - 1)
```

The subtraction accounts for the first token already included in TTFT. Some tools use slightly different conventions, so always state the exact definition used.

### Throughput

Total work completed by the server across all users, commonly measured as output tokens per second or requests per second.

High throughput does not guarantee low latency for an individual user. Batching more requests can increase total throughput while slowing each request.

## 3. Why Memory Bandwidth Matters

**Memory bandwidth** is the amount of data a GPU can move from its main memory to its compute units per second.

At low-batch decode, model weights must be read for each token step. The GPU may finish the required mathematics faster than memory can deliver those weights. In this regime, peak Floating-Point Operations Per Second (FLOPS) is a poor predictor of token-generation speed; achieved memory bandwidth is more useful.

This explains why:

- Reducing weight size can accelerate decode
- Batching can improve hardware utilization by reusing each weight read
- A GPU with higher memory bandwidth can outperform one with more theoretical compute
- Adding GPUs gives diminishing returns when communication overhead and smaller per-GPU transfers reduce efficiency

## 4. Model Bandwidth Utilization

**Model Bandwidth Utilization (MBU)** estimates how effectively a memory-bound inference system uses the GPU's available memory bandwidth.

```text
Achieved bandwidth ~= (model bytes + KV cache bytes read per step) / TPOT

MBU = achieved bandwidth / peak GPU memory bandwidth
```

### Worked Example

Assume:

- A 7-billion-parameter model
- 16-bit weights, using approximately 14 gigabytes
- TPOT of 14 milliseconds
- GPU peak memory bandwidth of 2 terabytes per second

Ignoring the KV cache for this simplified example:

```text
Achieved bandwidth = 14 GB / 0.014 seconds = 1 TB/s
MBU = 1 TB/s / 2 TB/s = 50%
```

An MBU of 50% means the server is achieving roughly half of the GPU's theoretical memory bandwidth. Values nearer 100% suggest less remaining room for software optimization in the memory-bound regime.

**Model FLOPS Utilization (MFU)** is the complementary measure for compute-bound workloads. MBU is most useful for low-batch decode; MFU is more useful after larger batches push the workload toward the compute limit.

## 5. Batching Strategies

| Method | Definition | Best fit | Main limitation |
|---|---|---|---|
| Static batching | The client submits a fixed group of requests together | Uniform offline workloads | Fast requests wait for the slowest request |
| Dynamic batching | The server groups incoming requests before execution | Short, similar requests | Performs poorly with varied lengths or settings |
| Continuous batching | The server adds and removes requests at each generation step | Shared online serving | More scheduling complexity |

Continuous batching improves GPU utilization because finished sequences leave immediately and new work enters without waiting for the entire batch.

### The Batch-Size Tradeoff

Increasing batch size usually:

- Raises total throughput
- Raises arithmetic intensity
- Increases per-request latency
- Increases KV cache memory usage

After the workload becomes compute-bound, a larger batch may increase latency without producing meaningful throughput gains. The useful operating point is the knee of the throughput-latency curve, not the largest batch that fits.

## 6. Quantization

**Quantization** stores weights or activations using fewer bits.

- 16-bit to 8-bit roughly halves weight memory and bytes moved
- 16-bit to 4-bit roughly quarters them
- Smaller weights can improve memory-bound decode and allow larger KV caches
- KV cache quantization can increase concurrent-request capacity

Quantization is not a free speed setting. It can reduce output quality, and its effect varies by model architecture and size. Benchmark the complete inference system using both performance metrics and a fixed quality evaluation.

## 7. Benchmarking Lessons

1. Define the workload before defining a latency target.
2. Report exact input and output token lengths.
3. Keep TTFT and TPOT separate.
4. Report throughput alongside per-user latency.
5. Sweep request rate or concurrency to expose the saturation curve.
6. Record batch size, hardware, engine version, model revision, and numeric precision.
7. Account for tokenizer differences before comparing tokens per second across model families.
8. Measure end to end because theoretical hardware limits do not include software overhead or inter-GPU communication.

Output length often dominates end-to-end latency because output tokens are generated sequentially. Input length may have a smaller latency effect but still raises memory requirements through the KV cache.

## Key Points to Retain

- Prefill is usually compute-bound; low-batch decode is usually memory-bandwidth-bound.
- TTFT measures when a response begins; TPOT measures how fast it continues.
- Throughput and per-user latency trade against each other.
- Model Bandwidth Utilization (MBU) measures bandwidth efficiency during memory-bound inference.
- Batching improves utilization until compute saturation, after which added batch size mostly adds latency.
- Quantization must be evaluated for both performance and quality.
- Hardware should be selected for the model architecture and real workload, not from peak specifications alone.

## Source Caution

The article's benchmark numbers use older models and hardware configurations. Keep the principles and measurement methods, but do not reuse its performance numbers as expectations for a current model, engine, or cloud GPU.
