# Week 4 Takeaways: SGLang and Batching

2026-11-22. The readings and SGLang deployment are complete. The shared-prefix comparison with vLLM is still open. [Companion deployment and Bruno runbook](https://github.com/crypticseeds/inference-engineering/tree/main/variants/sglang)

## What the Readings Explain

| Reading | What to remember |
|---|---|
| [RadixAttention announcement](https://lmsys.org/blog/2024-01-17-sglang/) and [SGLang paper](https://arxiv.org/abs/2312.07104) | SGLang stores reusable KV cache entries in a radix tree keyed by token sequences. A request finds the longest cached token prefix and only computes KV entries for the remaining prompt tokens. Shared system prompts are a good test case because they are repeated at the start of many requests. |
| [Efficient LLM Inference with SGLang](https://www.youtube.com/watch?v=Ny4xxErgFgQ) | The SGLang programming frontend and serving runtime are separate pieces. This week's deployment used the runtime through its OpenAI-compatible HTTP API. |
| [Continuous batching](https://www.anyscale.com/blog/continuous-batching-llm-inference) and [Orca](https://www.usenix.org/conference/osdi22/presentation/yu) | Iteration-level scheduling lets finished requests leave and waiting requests join between model iterations. With mixed output lengths, short requests can free slots while longer ones keep generating. |
| [Sarathi-Serve](https://arxiv.org/abs/2403.02310) | Chunked prefill splits a long prompt into smaller units of work. The scheduler can batch a prefill chunk with ongoing decode so a long incoming prompt is less likely to stall other users' token streams. Chunk size affects both prompt processing latency and decode interruptions. |
| [SGLang v0.4 scheduler](https://lmsys.org/blog/2024-12-04-sglang-v0-4/) | CPU scheduling work can overlap with GPU execution to reduce gaps between GPU steps. This addresses a separate source of overhead from prefix reuse and batching. |

PagedAttention and RadixAttention describe different parts of serving. PagedAttention manages the physical storage of KV cache blocks. RadixAttention indexes cached token prefixes for reuse across requests. vLLM can also reuse prefixes, so a fair comparison records whether caching is enabled on both engines and how the cache was primed. [Week 2 block-table notes](week2-vllm-architecture-and-block-table.md) · [vLLM prefix caching design](https://docs.vllm.ai/en/latest/design/prefix_caching/)

## What I Built and Verified

I ran SGLang 0.5.9 on one Modal A10 with the same pinned Qwen3-8B BF16 weights and 8,192-token context limit as the vLLM baseline. `modal serve` kept the HTTPS endpoint available while I tested from Bruno. Modal authenticated the requests and forwarded them to SGLang on the GPU. `GET /v1/models` returned `qwen3-8b` with `owned_by: sglang`. A non-streaming `POST /v1/chat/completions` returned an assistant reply with 22 prompt tokens, 50 completion tokens, and `finish_reason: stop`.

I saw intermittent failures during testing and did not capture a failed response together with the server logs. The server can scale to zero after two minutes idle, so cold starts are a possible explanation. The final logs show shutdown and an exit code of 0.

## Open Comparison and Week 5

The Bruno request verified that the API works. The shared-prefix TTFT comparison is deferred until after the Week 5 concurrent session. Revisit using GuideLLM for it then. Keep the two containers warm and use the same client, model revision, prompt and output settings on both. With prefix caching enabled, measure streaming TTFT for a long prefix the engine has never seen, then prime that prefix and measure requests that reuse it with different suffixes. Record repeat counts, TTFT percentiles, and cache hits where available. Keep GPU cold starts out of those measurements.

Week 5 starts with the GuideLLM introduction and install on 2026-11-23. The first sweep is the fixed-length vLLM baseline, followed by a `vllm bench serve` cross-check. [Progress and Week 5 checklist](../progress.md#week-5-load-testing)
