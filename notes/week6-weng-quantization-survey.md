# Week 6 Wed: Quantization Survey, and Which Engine Implements What

Sources: Quantization section of [Large Transformer Model Inference Optimization](https://lilianweng.github.io/posts/2023-01-10-inference-optimization/) (Lilian Weng, 2023-01-10), plus the [vLLM quantization docs](https://docs.vllm.ai/en/latest/features/quantization/) and [SGLang quantization docs](https://docs.sglang.io/docs/advanced_features/quantization) (latest) for the engine mapping.

## Main Idea

Generic 8-bit quantization works for CNNs but breaks on large transformers, because a few activation channels carry huge outliers. Every LLM-specific method is a way around those outliers: keep them in high precision, quantize at finer granularity, compensate errors with second-order information, or move the difficulty from activations into weights.

## 1. PTQ vs QAT

- **Post-training quantization (PTQ):** quantize a trained model, cheap.
- **Quantization-aware training (QAT):** quantize during training or fine-tuning, better quality, needs compute and representative data.
- Speedups need kernels for the mixed formats (for example INT4 x FP16). A format without a fast kernel only saves memory.

## 2. Why Transformers Are Hard

- FFN inputs and outputs have very different dynamic ranges, driven by strong outliers, so per-tensor quantization loses precision.
- Above about **6.7B parameters** (OPT), extreme outlier features appear in **all** layers: a few activation dimensions are about **100x** larger than the rest.

## 3. The Families of Fixes

| Fix | Method | Idea |
|---|---|---|
| Keep outliers in high precision | GOBO, Bondarenko et al., **LLM.int8()** | LLM.int8() quantizes row and column vectors separately to INT8 and keeps outlier feature dimensions in FP16. They are a tiny fraction of values |
| Finer granularity | Q-BERT (group per attention head), PEG (groups along the embedding dimension, with outliers sorted into the same group), ZeroQuant (group-wise weights, token-wise activations, fused kernels) | Scales that fit the local range |
| Second-order information | Q-BERT / HAWQ, **GPTQ** | HAWQ gives layers with large Hessian eigenvalues more bits. GPTQ minimises `‖WX − ŴX‖`, quantizing weights greedily and updating the remaining ones with the Hessian to compensate the error. OPT-175B to **3 or 4 bits** with little loss, weights only |
| Smooth the outliers away | **SmoothQuant** | `Y = (X diag(s)^-1)(diag(s) W)`: scale activations down and weights up by the same factor so both quantize well. `s_j = max|X_j|^α / max|W_j|^(1-α)`, with **α = 0.5** a good default. Enables **W8A8** |
| Train for it | QAT, distillation, layer-by-layer distillation (LKD) | Recover quality with training |

AWQ (2023) postdates the survey. It is closest to SmoothQuant's insight applied to weight-only 4-bit: find the weight channels that multiply large activations and scale them up before quantizing so they keep more precision.

## 4. Which Engine Implements What (the Wednesday task)

### vLLM hardware support (from its docs)

| Method | Ampere (A10, A100) | Ada (L4, L40S) | Hopper (H100) |
|---|:---:|:---:|:---:|
| AWQ | ✅ | ✅ | ✅ |
| GPTQ | ✅ | ✅ | ✅ |
| Marlin kernels (GPTQ / AWQ / FP8 / FP4, weight-only) | ✅ | ✅ | ✅ |
| LLM Compressor INT8 W8A8 | ✅ | ✅ | ✅ |
| **LLM Compressor FP8 W8A8** | ❌ | ✅ | ✅ |
| bitsandbytes, GGUF | ✅ | ✅ | ✅ |

Plus quantized KV cache (`--kv-cache-dtype fp8`), ModelOpt, AMD Quark and online FP8 quantization (`--quantization fp8`).

**This table is why Week 6 ran on the L40S.** On an A10, FP8 W8A8 is not supported; an FP8 checkpoint falls back to Marlin weight-only, with 16-bit compute.

### SGLang (latest docs)

- Loads pre-quantized checkpoints from the model config: AWQ, GPTQ, FP8 and others.
- Online `--quantization` values include `fp8`, `w8a8_fp8`, `w8a8_int8`, `modelopt_fp8` (Hopper+), `modelopt_fp4` (Marlin W4A16 fallback before Blackwell) and AMD-specific formats.
- Plain `gptq` is removed on NVIDIA in favour of `gptq_marlin`; `awq_marlin` and `gptq_marlin` are CUDA-only.
- Recommends offline quantization with LLM Compressor, ModelOpt, GPTQModel or auto-round, and validating every quantized model with benchmarks.

### What I ran

| Checkpoint | Method family | Storage / compute | vLLM kernel on the L40S |
|---|---|---|---|
| `Qwen/Qwen3-8B-FP8` | Block-wise FP8 E4M3, one weight scale per 128 x 128 block, dynamic activation scales | W8A8 FP8 | Block FP8 W8A8 (default config, no L40S tuning) |
| `Qwen/Qwen3-8B-AWQ` | AWQ, 4-bit, group size 128, with zero points (asymmetric) | W4A16 | `awq_marlin` |

## Key Points to Retain

- The LLM problem is activation outliers, emerging around 6.7B parameters.
- Four fixes: keep outliers in FP16 (LLM.int8()), finer groups (ZeroQuant), Hessian error compensation (GPTQ), move difficulty into weights (SmoothQuant; AWQ for weight-only).
- Weight-only (W4A16, GPTQ, AWQ) saves memory and helps memory-bound decode. Weight-and-activation (W8A8 INT8 or FP8) also speeds up compute-bound prefill and large batches, but needs hardware support: FP8 needs Ada or Hopper.
- Check the engine's hardware table before choosing a GPU for a quantization test.
