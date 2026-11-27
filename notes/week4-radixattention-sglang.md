# Week 4 Mon: RadixAttention and SGLang

Source: [Fast and Expressive LLM Inference with RadixAttention and SGLang](https://lmsys.org/blog/2024-01-17-sglang/) (LMSYS, 2024-01-17).

## Main Idea

LLM applications rarely make one call. Agents, few-shot prompts, tree search and multi-turn chat send many requests that **share a prefix**: the same system prompt, the same examples, the same conversation so far. A standard engine throws away the Key-Value (KV) cache when a request finishes, so every request recomputes that shared prefix.

SGLang keeps it. **RadixAttention** retains the KV cache of finished prompts and generations and reuses it automatically for any later request that starts with the same tokens. No manual configuration of which prefixes to cache.

## 1. How RadixAttention Works

| Piece | What it does |
|---|---|
| Radix tree | Maps token sequences (keys) to their KV cache tensors (values). A radix tree is a compact trie: one edge can hold a whole run of tokens, not just one |
| Where it lives | The tree structure is on the CPU (cheap to maintain). The KV tensors are on the GPU in a paged layout where **one page = one token** |
| Matching | The frontend always sends the full prompt. The runtime finds the longest cached prefix, reuses it and only runs prefill for the rest |
| Eviction | LRU: when GPU memory runs out, evict the least recently used **leaf** nodes, recursively |
| Scheduling | A cache-aware scheduling policy orders waiting requests to raise the hit rate (details in the [paper note](week4-sglang-paper.md)) |

It composes with continuous batching and paged attention rather than replacing them, and extends to image tokens for multimodal models.

```text
system prompt ── few-shot examples ──┬── question A ── answer A
                                     ├── question B ── answer B
                                     └── question C   (new request: only this suffix is prefilled)
```

## 2. The Frontend Language

SGLang is also a Python-embedded language for writing LLM programs (syntax inspired by Guidance). It runs in interpreter mode or compiler mode (traced into a dataflow graph).

| Primitive | Meaning |
|---|---|
| `gen` | Generate and store the result in a variable. Non-blocking, so several calls run in parallel |
| `fork` | Make parallel copies of the prompt state |
| `[var]` | Read a generated result (blocks until ready) |
| `choices` | Constrain the generation to a set of options |
| `run` | Execute an SGLang function |

Frontend and runtime are co-designed: `fork` tells the runtime a shared prefix is coming, which is exactly what RadixAttention wants to know. This week I only used the runtime, through its OpenAI-compatible API.

## 3. The Benchmark

| Item | Value |
|---|---|
| Models | Llama-7B on 1 A10G (24 GB), Mixtral-8x7B on 8 A10Gs with tensor parallelism, FP16 |
| Baselines | vLLM v0.2.5, Guidance v0.1.8, Hugging Face TGI v1.3.0 |
| Workloads | MMLU (5-shot), HellaSwag (20-shot), ReAct agent, tree-of-thought on GSM-8K, JSON decode, short and long multi-turn chat, DSPy RAG, LLaVA bench |

Results:

- **Up to 5x higher throughput** than Guidance and vLLM across these tasks.
- Lower latency, most of all **first-token latency**, where a prefix hit skips most of the prefill.
- An ablation found **no noticeable overhead without cache hits**, so RadixAttention is always on in the runtime.

Note the workload mix. Every benchmark above is prefix-heavy by construction (few-shot examples, multi-turn history, branching search). The 5x tells you which workloads it wins on, not what a single-shot chat API gains.

## 4. Future Work Named

Multi-layer storage (spilling cache beyond GPU memory) and better eviction policies.

## Key Points to Retain

- RadixAttention is **automatic prefix reuse across requests**: keep the KV cache after a request ends and look it up by token prefix.
- Tree on CPU, KV on GPU in one-token pages, LRU eviction of leaves.
- The win is mostly TTFT, because prefill is the work that gets skipped.
- Gains depend on shared prefixes. Benchmark with a workload that has them, and one that does not, before claiming anything.
- For a fair vLLM comparison, vLLM's own prefix caching must be on too (both engines had it on in my runs).
