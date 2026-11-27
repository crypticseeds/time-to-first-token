# Week 5 Wed: `vllm bench serve`

Source: [vLLM Benchmark CLI](https://docs.vllm.ai/en/latest/benchmarking/cli/) (vLLM docs, `latest`).

## Main Idea

`vllm bench serve` is vLLM's own **online serving benchmark**. It sends requests to a running server and measures latency and throughput **at the client**. A second tool with a different client and request generator is a cross-check: if it agrees with GuideLLM, the result is not an artifact of one client. If it disagrees, the difference is information.

## 1. Basic Use

```bash
vllm serve NousResearch/Hermes-3-Llama-3.1-8B

vllm bench serve \
  --backend vllm \
  --model NousResearch/Hermes-3-Llama-3.1-8B \
  --endpoint /v1/completions \
  --dataset-name sharegpt \
  --dataset-path ShareGPT_V3_unfiltered_cleaned_split.json \
  --num-prompts 10
```

## 2. Load Control

| Flag | Meaning |
|---|---|
| `--request-rate` | Target req/s. `inf` sends everything at once (maximum throughput) |
| `--burstiness` | Gamma-distribution shape for arrivals. Lower = burstier, higher = more uniform. Only applies with a finite rate |
| `--max-concurrency` | Cap on outstanding requests. Unlimited if unset |
| `--num-prompts` | How many requests in total |

Difference from GuideLLM: it sends a **fixed number of requests and waits for all of them** (drain), while my GuideLLM stages stopped at a time limit. At saturation the two measure slightly different things.

## 3. Workload

- Datasets: ShareGPT, BurstGPT, `random` (synthetic), prefix repetition, many Hugging Face sets, and custom JSONL.
- `--random-input-len`, `--random-output-len`: fixed synthetic lengths.
- `--ignore-eos`: keep generating to the requested length, so every request has the same output size.

## 4. Metrics

| Metric | Definition |
|---|---|
| TTFT | Request sent to first streamed output |
| ITL | Gap between consecutive streamed outputs, aggregated over all gaps |
| TPOT | Per request: `(e2e latency - TTFT) / (output tokens - 1)`, then aggregated over requests |
| E2EL | End-to-end request latency |
| Throughput | Requests/s, output tokens/s, total tokens/s |

`--percentile-metrics` picks which metrics get percentiles, `--metric-percentiles` which percentiles (for example `50,95,99`), `--save-result` writes JSON. I also used `--goodput ttft:500`: the rate of requests that met a 500 ms TTFT target, which turns an SLO into one throughput number.

Reminder from week 3: this tool drops one-token requests from TPOT, while vLLM's Prometheus TPOT counts them as 0.

## 5. Other Subcommands

`vllm bench throughput` (offline, no server), `vllm bench sweep`, `vllm bench startup` (time to first answer from a new server) and per-stage benchmarks for multimodal processing.

## How My Cross-Check Went

Same A10, same engine flags, seed 42, 72 formatted input and 16 output tokens with `--ignore-eos`, near-uniform arrivals (`--burstiness 10000`), concurrency cap 64. Run over loopback **inside** the serving container.

| | 16 req/s | 32 req/s |
|---|---:|---:|
| GuideLLM p95 TTFT (from the Pi) | 307.2 ms | 2,514.6 ms |
| `vllm bench serve` p95 TTFT (loopback) | 128.8 ms | 2,442.9 ms |
| `vllm bench serve` goodput at 500 ms | 15.82 req/s | 0.23 req/s |

Same verdict: pass at 16, fail at 32, saturating near 20.5 req/s completed. The 180 ms gap at 16 req/s is the network between the Pi and Modal, which loopback removes.

## Key Points to Retain

- Two tools agreeing on the **verdict** is the cross-check. Absolute TTFT differs with client location.
- Drain-based (fixed count) and time-window runs differ at saturation. Say which you used.
- `--goodput` expresses an SLO as req/s directly.
- `--ignore-eos` plus fixed random lengths is the way to pin the workload.
