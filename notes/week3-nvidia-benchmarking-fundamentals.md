# Week 3 Thu (part 2): NVIDIA LLM Inference Benchmarking Fundamentals

Source: [LLM Inference Benchmarking: Fundamental Concepts](https://developer.nvidia.com/blog/llm-benchmarking-fundamental-concepts/), NVIDIA, April 2025. (The README link `llm-inference-benchmarking-fundamental-concepts/` now 404s; this is the current URL.)

## Main Idea

Every benchmarking tool reports "TTFT", "ITL" and "tokens per second", but they **define and measure them differently**. Numbers from GenAI-Perf, LLMPerf and `vllm bench` are not directly comparable unless you know each tool's formula. The post pins down the definitions and the load parameters that change them.

## 1. Load Testing vs Performance Benchmarking

| | Load testing | Performance benchmarking |
|---|---|---|
| Question | Can the deployment survive real traffic? | How efficiently does the model serve? |
| Finds | Capacity, autoscaling, network, resource issues | Model efficiency, optimization, config issues |
| Week in roadmap | Week 5 (1000 concurrent) and week 8 | Week 3 to 5 sweeps |

You need both.

## 2. Vocabulary

- **Input Sequence Length (ISL):** tokens the model receives, including system prompt, chat history, Retrieval-Augmented Generation (RAG) documents.
- **Output Sequence Length (OSL):** tokens the model generates.
- **Context length:** ISL plus output so far, at each step.
- Roughly 0.75 English words per token, but tokenizers differ per model.

## 3. Metric Definitions (and Where Tools Differ)

### Time to First Token (TTFT)

Includes **queueing + prefill + network latency**. Grows with ISL. GenAI-Perf and LLMPerf skip empty first chunks (no token) when timing.

### End-to-end latency

```text
e2e_latency = TTFT + generation_time
```

`generation_time` is first token received to last token received.

### Inter-Token Latency (ITL), also called TPOT

```text
ITL = (e2e_latency - TTFT) / (total_output_tokens - 1)
```

- GenAI-Perf **excludes** TTFT from ITL, so ITL describes decode only.
- LLMPerf **includes** TTFT. Same name, different number.
- ITL rises slowly with context length because KV cache and attention cost grow. Steady ITL means good memory management.

### Tokens Per Second (TPS)

| Variant | Formula | Note |
|---|---|---|
| TPS per system (GenAI-Perf) | `total_output_tokens / (T_last_response - T_first_request)` | Uses a sliding window, drops warm-up and cool-down |
| TPS per system (LLMPerf) | `total_output_tokens / (T_end - T_start)` | Includes prompt generation and bookkeeping; up to 33% of duration at concurrency 1 |
| TPS per user | `OSL / e2e_latency` | Approaches `1 / ITL` as OSL grows |

As concurrency rises, **system TPS rises until saturation (then may fall), while per-user TPS falls.**

### Requests Per Second (RPS)

```text
RPS = completed_requests / (T_last_response - T_first_request)
```

## 4. Workload Shapes (ISL / OSL)

| Use case | ISL | OSL | Stresses |
|---|---|---|---|
| Translation | ~500-2000 | ~500-2000 | Both |
| Generation (code, email) | O(100) | O(1000) | Decode, ITL |
| Summarization, RAG, multi-turn | O(1000) | O(100) | Prefill, TTFT |
| Reasoning | O(100) | O(1000-10000) | Decode, KV memory |

Longer ISL raises TTFT. Longer OSL raises ITL and KV memory. Benchmark with your real distribution.

## 5. Load Control Parameters

- **Concurrency N:** exactly N requests in flight; a new one starts as each finishes. NVIDIA's recommended knob. (LLMPerf sends batches and drains to zero between them, so its effective concurrency sags.)
- **Request rate:** constant (one every `1/r` s) or Poisson. Risk: if the rate exceeds capacity, outstanding requests grow without bound.
- **Max batch size:** what the engine processes at once. When `concurrency > max_batch x replicas`, extra requests queue and **TTFT climbs** while throughput flattens.
- **Sweep concurrency from 1 to a bit above max batch size.** Throughput saturates near max batch size; past that you only measure queueing.

## 6. Other Parameters That Skew Results

- **`ignore_eos=True`** for benchmarks so every request hits the target OSL. Otherwise output length is up to the model and runs are not comparable.
- **Sampling method** (greedy vs top-p/top-k/temperature) changes per-token cost. Keep it fixed across runs.

## Key Points to Retain

- Same metric name, different formula per tool. Always state the tool and definition.
- TTFT includes queueing and network, not just prefill.
- ITL should exclude the first token.
- System throughput and per-user throughput move in opposite directions under load.
- Concurrency above max batch size measures queueing, not the engine.
- Set `ignore_eos` and fix sampling for reproducible runs.
