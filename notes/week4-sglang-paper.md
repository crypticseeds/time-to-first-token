# Week 4 Wed: SGLang Paper and Talk

Sources: [SGLang: Efficient Execution of Structured Language Model Programs](https://arxiv.org/abs/2312.07104) (NeurIPS 2024) and the talk [Efficient LLM Inference with SGLang](https://www.youtube.com/watch?v=Ny4xxErgFgQ) (Lianmin Zheng, SGLang lead). These notes follow the paper; the talk presents the same system.

## Main Idea

Real applications are **LM programs**: many generation calls with control flow, parallel branches and structured output. They are hard to write (string handling, brittle parsing, manual parallelism) and slow to run, because engines do not see the program. Two concrete wastes:

1. **KV cache is discarded** after each request, so calls sharing a prefix recompute it.
2. **Constrained decoding goes one token at a time**, even when the grammar leaves only one valid continuation.

SGLang fixes both with a frontend language and a runtime designed together.

## 1. Frontend Primitives

| Primitive | Meaning |
|---|---|
| `extend` / `+=` | Append to the prompt |
| `gen` | Generate into a named variable; `regex=` constrains the output |
| `select` | Pick the highest-probability option from a list |
| `fork` / `join` | Split the prompt state into parallel branches and merge them |
| `image`, `video` | Multimodal inputs |

**Interpreter mode** (default) runs each prompt as an async stream in a background thread, so independent calls overlap. **Compiler mode** traces the program into a graph for static optimizations.

## 2. RadixAttention in Detail

- LRU cache of prompt and generation KV in a **radix tree**; KV in a paged layout with one token per page, sharing the memory pool with running requests.
- **Reference counters** per node: a node is evictable only when no running request uses it.
- On `fork`, the frontend sends the shared prefix first as a hint, so it lands in the tree once.

### Cache-aware scheduling

- Cache hit rate = cached prompt tokens / prompt tokens.
- Waiting requests are sorted by **matched prefix length**, longest first, instead of first-come-first-served.
- Theorem 3.1: for a batch, visiting the radix tree in **depth-first order** gives the optimal hit rate when the cache is at least the maximum request length. Longest-shared-prefix-first is equivalent to depth-first.
- Cost: requests with no shared prefix can **starve**. The paper lists fixing this as future work.

### Multi-GPU

- **Tensor parallel:** each GPU holds a shard of the KV cache; tree operations are identical, so no extra sync.
- **Data parallel:** each worker has its own sub-tree; a router keeps a meta-tree and does prefix matching there. (This became the cache-aware router in [v0.4](week4-sglang-v0-4-scheduler.md).)

## 3. Compressed Finite State Machine

A regex (such as a JSON schema) becomes a finite state machine. Normal constrained decoding masks invalid tokens at every step. SGLang **merges runs of single-choice transitions** into one compressed edge and decodes the whole run in one forward pass (**jump-forward decoding**). It retokenizes the text afterwards so token boundaries stay correct.

## 4. API Speculative Execution

For API-only models, SGLang lets the first call keep generating past its stop condition and reuses the extra output for later primitives, saving round trips and repeated input-token charges.

## 5. Evaluation

| Item | Value |
|---|---|
| Models | Llama-2 7B and 70B, Mixtral-8x7B, LLaVA-v1.5-7B, LLaVA-NeXT-34B, GPT-3.5 (API). FP16 |
| Hardware | AWS A10G (24 GB), tensor parallel for larger models; extra runs on A100 80 GB |
| Baselines | vLLM v0.2.5, Guidance v0.1.8, LMQL v0.7.3, HF Transformers for multimodal |
| Workloads | MMLU, HellaSwag, ReAct, generative agents, tree-of-thought, skeleton-of-thought, LLM judges, JSON decoding, multi-turn chat, DSPy RAG |

| Result | Number |
|---|---|
| Throughput | Up to **6.4x** vs the baselines |
| Latency | Up to **3.7x** lower |
| Multimodal throughput | Up to **6x** |
| Cache hit rate across benchmarks | **48% to 99%**; scheduling reaches about **96%** of optimal on average |
| Chatbot Arena production hit rate | **52.4%** (LLaVA-NeXT-34B), **74.1%** (Vicuna-33B), first-token latency **1.7x** lower on Vicuna-33B |
| Tree overhead | Under **0.3%** (0.2 s of 74.3 s for 100 ShareGPT requests) |
| Compressed FSM | **1.6x** JSON decoding throughput |

## 6. Limitations and Future Work

More output modalities; RadixAttention across the memory hierarchy (DRAM, disk); fuzzy semantic matching; higher-level primitives; fixing starvation in cache-aware scheduling; smarter compiler; compressed FSM can distort token probabilities.

## Key Points to Retain

- Two separate ideas: **reuse the prefix** (RadixAttention) and **skip forced tokens** (compressed FSM).
- Longest-prefix-first scheduling buys hit rate at the risk of starvation.
- The production numbers (52 to 74% hit rate, 1.7x faster first token) are the realistic expectation; the 6.4x comes from prefix-heavy programs.
- The tree costs almost nothing when there are no hits, which is why SGLang leaves it on.
