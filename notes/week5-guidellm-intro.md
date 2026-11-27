# Week 5 Mon: GuideLLM Introduction

Source: [GuideLLM: Evaluate LLM deployments for real-world inference](https://developers.redhat.com/articles/2025/06/20/guidellm-evaluate-llm-deployments-real-world-inference) (Red Hat Developer, 2025-06-20, updated 2026-03-10).

## Main Idea

Going from a proof of concept to production with real Service-Level Objectives (SLOs) means answering sizing questions: which model, which GPU, how many replicas, at what load does latency break. GuideLLM answers them by **simulating realistic traffic** against an OpenAI-compatible endpoint and reporting the latency and throughput distribution at each load level.

The key design choice: a **sweep**, not one load point. One arbitrary request rate tells you nothing about where the deployment breaks.

## 1. What It Is For

| Use case | Question it answers |
|---|---|
| Pre-deployment benchmarking | What are this model's limits on this hardware? |
| Regression testing | Did the new engine or model version get slower? |
| Hardware evaluation | Which GPU or instance gives the most throughput for the money? |
| Capacity planning | How many servers for the projected load? |

## 2. How It Works

- **Backend:** OpenAI-style HTTP, so it works with vLLM, SGLang or anything compatible.
- **Data:** a Hugging Face dataset, or **synthetic prompts** with set token counts (`synthetic:512,256` in the article's older syntax).
- **Scheduler with pluggable strategies:** constant rate, fixed concurrency, Poisson arrivals.

### The sweep

```text
1. synchronous   one request at a time          -> best-case latency
2. throughput    everything in parallel         -> maximum req/s
3. rates in between, spread across that range   -> the latency vs load curve
```

Each step runs about 30 seconds in the article's default.

## 3. Metrics

| Metric | Definition in the article |
|---|---|
| Requests per second | Completed requests per second: system throughput |
| Request latency | Start to finish for one request |
| Time to first token (TTFT) | Time to the first output token: what a chat user feels |
| Inter-token latency (ITL) | Average gap between consecutive tokens, **excluding the first** |
| Output tokens per second | Generated tokens per second across all requests |

All reported with percentiles, so you read p50 and the tail, not a mean.

## 4. Reading the Result Against an SLO

The article's worked example: a chat SLO of **TTFT under 200 ms for 99% of requests**. The sweep shows the deployment meets it up to about **10.69 req/s**, so one server handles about 10 req/s. Supporting 1,000 req/s then needs about **100 servers** of the same configuration.

That is the whole method: **SLO + sweep = capacity per server.**

## 5. Commands

```bash
pip install guidellm
vllm serve unsloth/Llama-3.1-8B-Instruct-bnb-4bit
guidellm --target http://localhost:8000/v1 --rate-type sweep --data "synthetic:512,256"
```

This is the 2025 CLI. GuideLLM 0.8.1, which I used, has `guidellm run` with `--backend`, `--profile`, `--data` and `--constraint` key=value options (see the [repo note](week5-guidellm-repo.md)). Output goes to `benchmarks.json`, YAML or CSV.

## 6. Accuracy Claims

The article reports **99.7%** load timing accuracy in internal tests, and **9.98** average concurrency when 10 was targeted. A load generator that cannot hold its target makes every number downstream wrong. I hit this myself at 32 req/s on the A10 (see the [build log](week5-build-log.md)).

## Key Points to Retain

- Sweep from synchronous to saturation; the curve is the result, not one point.
- Pick the SLO first (metric, percentile, threshold), then read the highest rate that meets it.
- ITL here excludes the first token. Same name as other tools, check the definition (week 3 lesson).
- Capacity per server x replicas = fleet size, but only for the workload you tested.
- Check that the client actually delivered the load you asked for.
