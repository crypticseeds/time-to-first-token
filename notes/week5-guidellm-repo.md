# Week 5 Tue: GuideLLM Repository

Source: [vllm-project/guidellm](https://github.com/vllm-project/guidellm) README. I used GuideLLM 0.8.1.

## Main Idea

The README is the operating manual: how to install, how to describe a load shape, how to feed it prompts, and where results go. Everything is configured as `kind=...,key=value` options, so a run is fully described by its command line.

## 1. Install and Run

```bash
pip install guidellm[recommended]
vllm serve "neuralmagic/Meta-Llama-3.1-8B-Instruct-quantized.w4a16"

guidellm run \
  --backend kind=openai_http,target=http://localhost:8000 \
  --profile kind=sweep \
  --constraint kind=max_duration,seconds=30 \
  --data kind=synthetic_text,prompt_tokens=256,output_tokens=128
```

A container image is also published at `ghcr.io/vllm-project/guidellm`, configured with `GUIDELLM__SPEC__*` environment variables.

## 2. Profiles: the Load Shape

| Profile | Parameter | What it does | When I used it |
|---|---|---|---|
| `synchronous` | - | One request at a time | Monday baseline |
| `throughput` | `max_concurrency=` | Everything at once, maximum req/s | - |
| `concurrent` | `streams=` | Fixed number of requests in flight (closed loop) | H100 sweep, 1 to 1,000 streams |
| `constant` | `rate=` (req/s) | Fixed arrival rate (open loop), optional `max_concurrency` cap | A10 rate sweep, 1 to 32 req/s |
| `poisson` | `rate=` | Random arrivals averaging `rate` | - |
| `sweep` | - | Synchronous, throughput, then rates in between | - |

Profiles also accept `warmup` and `cooldown` (fraction or absolute), for example `kind=concurrent,streams=16,warmup=0.1`.

**Open vs closed loop matters.** `constant` keeps sending at the rate whether or not the server keeps up, so queues grow and you see saturation. `concurrent` only sends when a slot frees, so it can never overload the server more than N in flight. They answer different questions.

## 3. Data

| Kind | Example |
|---|---|
| `synthetic_text` | `prompt_tokens=64,output_tokens=16` (my workload) |
| `huggingface` | `source=abisee/cnn_dailymail,load_kwargs.name=3.0.0` |
| `json_file`, `csv_file`, `text_file` | Local prompt files |
| `synthetic_image`, `synthetic_video` | Multimodal prompts |

**Fixed lengths are a deliberate choice.** Prompt and output length set the KV cache per request, which sets how many requests fit, which sets the whole curve. Fix them to make runs comparable; use a realistic distribution when you want a realistic answer (pre-publish checklist item).

Note: configured prompt tokens are not what the server sees. My 64 configured tokens arrived as **72** after the chat template.

## 4. Limits and Output

| Option | Meaning |
|---|---|
| `--constraint kind=max_duration,seconds=N` | Stop each stage after N seconds |
| `--constraint kind=max_requests,count=N` | Stop after N requests |
| `--output kind=json,path=...` | Report path; replaces the default `benchmarks.json` and `benchmarks.csv` |

Formats: JSON or YAML for analysis, CSV for spreadsheets, HTML for a standalone report, PLOT for static graphs. Reports carry full TTFT, ITL and end-to-end distributions.

## 5. Warnings Worth Reading

GuideLLM records **root dispatch delay**: how late the client sent each request against its schedule. Above **0.2 s** it warns that the stage did not run at its nominal load. This is a client-side metric, separate from TTFT. At the A10's 32 req/s stage, mine hit 18.9 s at p95: the stage measured saturation, not 32 req/s.

## Key Points to Retain

- A run = backend + profile + data + constraints. Record the whole command.
- `constant` (open loop) finds where the queue grows; `concurrent` (closed loop) maps throughput vs in-flight count.
- Fix lengths for comparability, and report formatted token counts, not configured ones.
- Check dispatch delay before trusting a stage.
