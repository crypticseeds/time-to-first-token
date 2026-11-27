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

## What I Did Not Do

The week's deliverable included a first prefix-reuse comparison against vLLM. I deferred it: a fair test needs both containers warm, the same client, and a long shared prefix primed and then reused. The week 5 harness (GuideLLM from a Modal client, both engines live in one dashboard) is the right tool for it, so it waits until after the 1,000-concurrent session.

Planned shape of that test:

1. Both engines warm, prefix caching on in both (`--enable-prefix-caching` on vLLM; RadixAttention is SGLang's default).
2. Streaming TTFT for a long prefix neither engine has seen.
3. Prime it, then send requests that share it with different suffixes.
4. Record repeat counts, TTFT percentiles and cached prompt tokens (`prompt_tokens_details.cached_tokens` in the response usage).
5. Keep cold starts out of the measurement.

## What Surprised Me

- How little changes between engines at the API level. Same OpenAI routes, same request body; only `owned_by` gives it away.
- That "same config" needs care: memory fractions, CUDA graphs and default sampling parameters are named differently in each engine, and each has to be checked rather than assumed.

## Next

Week 5: GuideLLM load testing. First a fixed-length vLLM sweep, then the same sweep on SGLang.
