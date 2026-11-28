# Week 6 Takeaways: Quantization

Quick recap of the week 6 readings. Full notes:

- [MIT 6.5940 Lecture 5, Quantization Part I](week6-mit-lecture5-quantization-i.md) (Mon)
- [MIT 6.5940 Lecture 6, Quantization Part II](week6-mit-lecture6-quantization-ii.md) (Tue)
- [Quantization survey and engine mapping](week6-weng-quantization-survey.md) (Wed)

What I built and measured: [Week 6 build log](week6-build-log.md).

## The One-Line Version

**Quantization trades bits for bytes: weight-only formats speed up memory-bound decode, weight-and-activation formats also speed up compute, and only a paired quality test on a real task tells you what it cost.**

## Format Cheat Sheet

| Format | Stored | Computed | Speeds up | Needs | My result on Qwen3-8B, L40S |
|---|---|---|---|---|---|
| BF16 | 16-bit weights | 16-bit | Baseline | Any modern GPU | 87.6% GSM8K, 15.3 GiB |
| FP8 W8A8 (E4M3, block scales) | 8-bit weights and activations | 8-bit | Decode and compute-bound batches | Ada or Hopper | 88.0% (no change), 8.8 GiB, most req/s at 250 concurrent |
| INT8 W8A8 (SmoothQuant) | 8-bit | 8-bit integer | Same, on Ampere too | INT8 kernels | Not tested |
| INT4 W4A16 (AWQ, GPTQ) | 4-bit weights, group scales | 16-bit after dequant | Memory-bound decode only | Marlin kernels | 85.6% (−2 pt), 5.7 GiB, fastest at 1 request, BF16-level at 250 |
| FP8 KV cache | 8-bit KV | Attention | Context length, batch size | `--kv-cache-dtype fp8` | Not tested |

## Ten Things to Remember

1. **Exponent bits set range, fraction bits set precision.** BF16 keeps FP32's range; FP8 E4M3 tops out at 448, so every FP8 tensor needs a scale.
2. **`r = S(q − Z)`.** Symmetric (Z = 0) for weights, asymmetric for skewed activations.
3. **Granularity is the main quality knob.** Per-tensor fails on outlier channels; per-channel fixes weights; groups of 128 make 4-bit work. Count the scales: 4-bit with a scale per 16 values is 4.25 bits.
4. **Calibrate activations on real samples** and clip to minimise MSE or KL, not to the absolute maximum.
5. **LLMs grow activation outliers** (around 6.7B parameters, about 100x the typical value). LLM.int8(), GPTQ, SmoothQuant and AWQ are four different ways around them.
6. **Weight-only vs weight-and-activation is the distinction that matters for speed.** W4A16 halves bytes again but still computes in 16 bits.
7. **Decode at low batch is bandwidth-bound:** time per token ≈ weight bytes ÷ memory bandwidth. AWQ hit 8.2 ms against a 7 ms floor; BF16 21.7 ms against 19 ms.
8. **The kernel matters as much as the format.** FP8 decode ran at 18.7 ms against an 11 ms floor because vLLM had no tuned FP8 config for the L40S.
9. **Use a paired test for quality.** Every variant answers the same questions, so count disagreements (McNemar). FP8 vs BF16: 29 to 24, p = 0.58. AWQ vs BF16: 46 to 73, p = 0.017.
10. **Check hardware support before choosing a GPU.** FP8 W8A8 needs Ada or Hopper; on an A10 it silently becomes weight-only.

## Deliverable Checklist

Roadmap deliverable: FP8 and INT4/AWQ variants benchmarked against the 16-bit baseline on throughput and a quality proxy.

- [x] BF16, FP8 and AWQ served on the same GPU type with the same engine settings
- [x] Quality table: GSM8K (paired test) and WikiText perplexity
- [x] Throughput and latency under the Week 5 concurrency sweep, with server-side metrics
- [x] Memory: weights and KV cache capacity per variant
- [ ] Not done, optional: a GPTQ variant (the roadmap asks for AWQ or GPTQ; Qwen publishes no official GPTQ build of Qwen3-8B), and a long-context run where the extra KV capacity matters (Week 7)

## Sources Note

- The roadmap links an older Lecture 5 video; these notes follow the Fall 2024 slides on the course page.
- The 2024 Lecture 6 slides do not cover AWQ, GPTQ or SmoothQuant. GPTQ and SmoothQuant come from the Weng survey; AWQ postdates it.
- Engine support tables come from the current vLLM and SGLang docs, which may list newer formats than vLLM 0.21 and SGLang 0.5.9.
