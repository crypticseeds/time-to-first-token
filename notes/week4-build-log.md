# Week 4 Build Log: SGLang on the Same Hardware

Date: 2026-11-22. One session from the Pi against Modal.
Code and runbook: [`inference-engineering/variants/sglang/`](https://github.com/crypticseeds/inference-engineering/tree/main/variants/sglang) and [`deploy/modal/sglang_qwen3_8b.py`](https://github.com/crypticseeds/inference-engineering/blob/main/deploy/modal/sglang_qwen3_8b.py).

## What I Built

- A Modal app that serves the same Qwen3-8B on SGLang, matched to the week 2 vLLM service so the engine is the only variable.
- A by-hand check of the OpenAI-compatible API from Bruno: model list and chat completion. The request order and body are in the runbook.

## Versions

| Component | Version |
|---|---|
| SGLang | 0.5.9 (`lmsysorg/sglang:v0.5.9`) |
| Model | `Qwen/Qwen3-8B` @ `b968826`, BF16, served as `qwen3-8b` (same revision as vLLM) |
| GPU | 1x NVIDIA A10 (24 GB) on Modal us-east, `max_containers=1` |
| SGLang flags | `--context-length 8192 --max-running-requests 16 --mem-fraction-static 0.85 --disable-cuda-graph` |
| vLLM baseline for comparison | 0.21.0, `--max-model-len 8192 --max-num-seqs 16 --gpu-memory-utilization 0.90 --enforce-eager` |

The limits were chosen to match vLLM: same context, same 16-request batch cap, CUDA graphs off on both. The memory settings are not identical by name: vLLM's 0.90 covers weights and KV cache, SGLang's 0.85 static fraction is its own accounting.

## Verification

| Check | Result |
|---|---|
| `GET /v1/models` | `qwen3-8b`, `owned_by: sglang`, `max_model_len: 8192` |
| `POST /v1/chat/completions` (non-streaming) | Assistant reply, 22 prompt tokens, 50 completion tokens, `finish_reason: stop` |
| Same request without credentials | HTTP 401: Modal's proxy auth was on |
| Final serving logs | Clean shutdown, exit code 0 |

## Setup Problems

| Problem | Status |
|---|---|
| Intermittent failed requests from Bruno during testing | **Unresolved.** I did not capture a failing response together with the server logs. The container scales to zero after 2 minutes idle, so cold starts are the likely cause. Next time: record Bruno's HTTP status next to the matching server logs |
| `GET /metrics` returned 404 | Found in week 5: SGLang needs `--enable-metrics`. Added to the launcher on 2026-11-24 |
| Modal proxy auth blocked Prometheus | Week 5 moved both apps to unauthenticated `modal serve` endpoints for scraping, with the hostname kept out of every committed file |

## Shared-Prefix Comparison (Buffer, Run 2026-11-27)

Deferred from the week itself until the week 5 harness existed. Full write-up: [A10 shared-prefix run](https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks/2026-11-27-modal-a10-prefix).

- Both engines on their own A10 with prefix caching on, a Modal client in us-east, raw 2,048-token prefix + 64-token suffix prompts, 32 output tokens.
- A probe first confirmed reuse: on a repeated prefix, both engines reported about 2,340 of about 2,360 prompt tokens as cached.
- `vllm bench serve` (`prefix_repetition`) as the main client, `sglang.bench_serving` (`generated-shared-prefix`) as the cross-check. 12 stages, 0 failed requests.

| TTFT p50 | vLLM | SGLang |
|---|---:|---:|
| Distinct prefixes, one at a time | 533 ms | 607 ms |
| Reused prefix (cache hit), one at a time | 73 ms | 170 ms |
| Distinct prefixes at 2 req/s | 4,863 ms (overloaded, 1.69 req/s completed) | 7,334 ms (overloaded, 1.61 req/s) |
| Reused prefixes at 2 req/s | 151 ms | 214 ms |

What it showed:

1. A cache hit removed 72 to 86% of one-request TTFT on 2,112-token prompts.
2. Reuse raised capacity too: 2 req/s overloaded both A10s with distinct prefixes and was easy with reused ones.
3. SGLang's hit TTFT has a floor about 100 ms higher than vLLM's on this setup. Not investigated.
4. This is two prefix caches against each other. The RadixAttention paper's gains are against engines without automatic prefix caching; vLLM 0.21 has it.

## What Surprised Me

- How little changes between engines at the API level. Same OpenAI routes, same request body; only `owned_by` gives it away.
- That "same config" needs care: memory fractions, CUDA graphs and default sampling parameters are named differently in each engine, and each has to be checked rather than assumed.

## Next

Week 5: GuideLLM load testing. First a fixed-length vLLM sweep, then the same sweep on SGLang.
