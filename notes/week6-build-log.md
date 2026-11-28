# Week 6 Build Log: Quantization Variants on L40S

Date: 2026-11-28. Pi as the control host, engines and clients on Modal.
Full write-up, raw data and dashboard: [L40S quantization run](https://github.com/crypticseeds/inference-engineering/tree/main/benchmarks/2026-11-28-modal-l40s-quantization).

## What I Built

- A `MODEL_VARIANT` switch in the vLLM launcher (bf16, fp8, awq) with pinned revisions, one Modal app per variant so all three serve side by side, and an `L40S` GPU profile.
- GPU placement pinned to us-east and logged at startup.
- An lm-evaluation-harness client on Modal for the quality proxy: GSM8K (5-shot, all 1,319 questions) and WikiText-2 perplexity.
- A `quant-concurrency` GuideLLM plan: 1, 10, 50, 100, 250 concurrent, the Week 5 workload.
- A Grafana dashboard with one line per variant, driven by a `variant` label on each Prometheus target.

## Versions

| Component | Version |
|---|---|
| vLLM | 0.21.0, 8,192 context, 256 running requests, CUDA graphs, 90% memory, BF16 KV cache for every variant |
| Checkpoints | `Qwen/Qwen3-8B` @ `b968826`, `Qwen/Qwen3-8B-FP8` @ `220b46e` (E4M3, 128 x 128 blocks), `Qwen/Qwen3-8B-AWQ` @ `4da05a8` (4-bit, group 128) |
| GPU | 1x NVIDIA L40S (48 GB, Ada) per variant, us-east |
| lm-evaluation-harness | 0.4.13 (`local-completions`), transformers 5.22.2 |
| GuideLLM | 0.8.1 |

## Results

| | BF16 | FP8 | AWQ INT4 |
|---|---:|---:|---:|
| GSM8K strict exact match | 87.6% | 88.0% | 85.6% |
| Change vs BF16 (paired McNemar) | | +0.4 pt, p = 0.58 | −2.0 pt, p = 0.017 |
| WikiText word perplexity | 13.48 | 13.53 (+0.3%) | 14.15 (+5.0%) |
| Weights | 15.3 GiB | 8.8 GiB | 5.7 GiB |
| KV cache capacity | 166,720 tokens | 213,232 (+28%) | 235,856 (+41%) |
| req/s at 1 concurrent | 2.6 | 3.2 | 5.9 |
| Inter-token latency at 1 concurrent | 21.7 ms | 18.7 ms | 8.2 ms |
| req/s at 250 concurrent (two runs) | 129 to 132 | 141 to 160 | 135 |

Zero request errors and zero preemptions in every stage.

## Issues and How I Resolved Them

| Issue | What I learned |
|---|---|
| **FP8 on the A10 would not be FP8.** Ampere has no FP8 tensor cores; vLLM's table lists FP8 W8A8 for Ada and Hopper only. Moved Week 6 to the L40S | Check the engine's hardware table before picking a GPU for a quantization test |
| **Serving three variants at once needed separate app names.** `MODEL_VARIANT` now picks the checkpoint and the app name | Make the variant part of the deployment identity |
| **lm-eval crashed on `No module named 'transformers'`.** The `[api]` extra does not install it; added it to the image | A `--limit 5` smoke test caught it on all three variants for the cost of one restart |
| **BF16 first-token latency was 258 ms against 40 ms for the others, while its inter-token latency was normal.** Its GPU had landed outside us-east. Pinned `compute_region="us-east"`, logged placement, reran: 40 ms | A first-token gap with normal inter-token latency is the network, not the engine |
| **The cloud could not be pinned.** Modal rejects `cloud="aws"` on both server and client. Measured the cross-cloud effect instead: at most 10 to 17 ms at concurrency 1, smaller than run-to-run variation | When a control is unavailable, measure and report the size of the effect |
| **Runs varied by up to 10%.** Ran the sweep three times and only claimed differences that held in both region-pinned runs | One run per variant cannot rank variants within 10% of each other |
| **The existing dashboard would merge the variants.** Its queries sum every vLLM target; the new one splits by a `variant` label | Put the comparison dimension in a Prometheus label |
| **Grafana showed no FP8 at first.** FP8 took 7.5 minutes to start and its sweep ran 5 minutes behind; Prometheus `up` history showed every stage was scraped | Check `up` and stage timestamps before assuming a scrape failure |
| **A Prometheus query read 0 req/s.** The success counter has one series per finish reason; `sum()` fixed it | Sum labelled counters |
| **vLLM warned the FP8 kernels might be sub-optimal** on the L40S (no tuned block-FP8 config). FP8 decode was 18.7 ms per token against an 11 ms bandwidth floor | Read startup warnings before trusting a throughput comparison |

## Findings

1. **FP8 is free on quality and wins at high load.** No measurable GSM8K or perplexity change, and the most throughput at 250 concurrent, where FP8 tensor cores do the compute-bound matmuls.
2. **AWQ wins only when decode is memory-bound.** 8.2 ms per token at one request, close to its 7 ms bandwidth floor and 2.6 times faster than BF16. At 250 concurrent it computes in 16 bits after dequantizing and lands at BF16's throughput.
3. **AWQ's quality cost is real.** 2 points on GSM8K with p = 0.017, and 5% on perplexity. That is above the roadmap's 1 to 2% bar, so INT4 is a memory-pressure lever here, not a default.
4. **Memory savings did not matter for this workload.** KV cache peaked at 13% with 88-token requests. Long contexts would be where FP8 and AWQ's extra KV capacity pays off.

## What Surprised Me

- How cleanly the physics showed up: inter-token latency at one request tracked weight bytes divided by memory bandwidth for BF16 and AWQ.
- That the biggest error in the first run was GPU placement, not anything about quantization.
- That FP8 decode was slower than its bytes suggest because of an untuned kernel. A quantization result depends on the kernel as much as the format.

Total GPU time: about 2.5 L40S-hours across three throughput runs and one quality run.

## Next

Week 7: speculative decoding and KV eviction. The long-context eviction test is also where FP8 KV cache and the larger quantized KV pools become relevant.
