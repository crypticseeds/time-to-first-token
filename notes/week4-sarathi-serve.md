# Week 4 Fri: Sarathi-Serve and Chunked Prefill

Source: [Taming Throughput-Latency Tradeoff in LLM Inference with Sarathi-Serve](https://arxiv.org/abs/2403.02310) (OSDI 2024).

## Main Idea

Continuous batching mixes prefill and decode in the same iterations, and they behave very differently:

| Phase | Work | Bound by | Latency it sets |
|---|---|---|---|
| Prefill | Whole prompt in parallel | **Compute** (linear layers are over 80% of runtime; nearly saturates with one request) | TTFT |
| Decode | One token per request per step | **Memory bandwidth** (reading weights from HBM) | Time Between Tokens (TBT) |

Schedulers had to pick a side:

- **Prefill-prioritizing** (vLLM, Orca): admit new prompts eagerly. High throughput, but a long prompt makes the whole iteration slow and every running stream freezes. These **generation stalls** can last seconds and wreck P99 TBT.
- **Decode-prioritizing** (FasterTransformer): finish the running batch before admitting prompts. Smooth TBT, but batches shrink as requests finish, so throughput drops.

Sarathi-Serve gets both with **chunked prefills** and **stall-free scheduling**.

## 1. The Key Observation

Decode iterations leave compute idle (low arithmetic intensity). So you can **piggyback prefill tokens** onto a decode batch almost for free, as long as you add only a bounded amount.

## 2. Chunked Prefills

Split a long prompt into chunks and process one chunk per iteration. Modest chunk sizes (from about 512 tokens) already saturate compute, so chunks lose little efficiency.

## 3. Stall-Free Batching

Each iteration, the scheduler builds the batch in this order, up to a **token budget** τ:

```text
1. all ongoing decodes (one token each)
2. the next chunk of any partially prefilled request
3. new requests' prefill tokens, only while under τ
```

Iteration time is now bounded by τ and nearly independent of how long incoming prompts are. Decodes never wait for a prefill to finish.

## 4. Choosing the Token Budget

| Pull toward smaller τ | Pull toward larger τ |
|---|---|
| Tighter TBT SLO | Chunking overhead: kernel launches, lower utilization, and re-reading earlier chunks' KV (the first chunk's KV is loaded N-1 times over N chunks) |
| Smaller pipeline bubbles | Fewer iterations per prompt, better TTFT |

**Tile quantization:** GPU matmuls run in tiles, so sizes that are not tile multiples waste work. A chunk of 257 tokens took **32% longer** to prefill than 256. Pick budgets on tile boundaries.

Overhead in practice: at most about **25%** extra prefill time at the smallest chunk (512), nearly nothing at 2,048.

## 5. Pipeline Parallel Bubbles

With pipeline parallelism, micro-batches of uneven compute leave stages idle. Falcon-180B example: a 4k-token prefill takes about 1,150 ms vs about 200 ms for a 32-request decode iteration, a **bubble of about 950 ms**. Uniform-size hybrid batches from chunking shrink those bubbles.

## 6. Evaluation

| Model | Hardware |
|---|---|
| Mistral-7B | 1x A100 80 GB |
| Yi-34B | 2x A100, TP2 |
| LLaMA2-70B | 8x A40, TP4-PP2 |
| Falcon-180B | 4x A100 per node x 2 nodes over 100 Gbps Ethernet, TP4-PP2 |

| Dataset | Median prompt / output tokens |
|---|---|
| `openchat_sharegpt4` | 1,730 / 415 |
| `arxiv_summarization` | 7,059 / 208 |

P99 TBT SLOs: strict 0.1 s (Mistral-7B), 0.2 s (Yi-34B), 1 s (70B, 180B). The relaxed SLOs are 5x those: 0.5 s, 1 s and 5 s.

Serving capacity under the SLO:

| Model | Gain |
|---|---|
| Mistral-7B | up to **2.6x** vs vLLM |
| Yi-34B | up to **3.7x** vs vLLM, **4.0x** vs Orca (strict SLO) |
| LLaMA2-70B | up to **4.3x** vs vLLM, **6.3x** vs Orca (strict SLO) |
| Falcon-180B | up to **5.6x** end-to-end; **3.6x** over vLLM's hybrid-parallel setup |

## 7. Limitations

- Chunks are slightly less efficient, so chunking alone can raise TTFT a little. Hybrid batching alone causes stalls. You need both.
- **Disaggregation** (Splitwise, DistServe: separate prefill and decode GPUs) avoids interference entirely and gives better TTFT, at the cost of moving KV cache between machines. That is week 8.

## Key Points to Retain

- Prefill is compute-bound, decode is memory-bound, so decode batches have spare compute for a bounded amount of prefill.
- Generation stalls are what a prefill-first scheduler does to P99 TBT when a long prompt arrives.
- Token budget τ is the knob: small for smooth streams, large for throughput and TTFT. Keep it a tile multiple.
- This is why chunked prefill is the default in both vLLM (`--max-num-batched-tokens` sets the budget) and SGLang.
- My week 5 prompts were 72 tokens, far below any chunk size, so chunking never engaged. Long-prompt runs are where to look for it.
