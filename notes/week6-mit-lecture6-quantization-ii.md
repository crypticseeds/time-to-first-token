# Week 6 Tue: MIT 6.5940 Lecture 6, Quantization Part II

Source: [MIT 6.5940 Lecture 6: Quantization Part II](https://youtu.be/wrcgWm_nUeE) (Song Han), Fall 2024 slides on the [course page](https://hanlab.mit.edu/courses/2024-fall-65940).

**Coverage note:** the roadmap pairs this session with AWQ, GPTQ and SmoothQuant. The 2024 Lecture 6 slides do not cover them: they stop at general post-training quantization, quantization-aware training, binary/ternary and mixed precision. In this course the LLM-specific methods come in a later lecture. GPTQ and SmoothQuant are in the [Weng survey note](week6-weng-quantization-survey.md); AWQ appears below from what my run used.

## Main Idea

Lecture 5 defined linear quantization. Lecture 6 asks how to choose its parameters (scale and zero point) well after training, and what to do when that is not enough.

## 1. Post-Training Quantization (PTQ): Three Questions

### Granularity: how many scale factors?

| Granularity | One scale per | Trade-off |
|---|---|---|
| Per-tensor | Whole weight tensor | Works for large models. Fails when output channels differ by more than 100x in range (outlier channels) |
| Per-channel | Output channel | The standard fix for weights |
| Group | Group of elements (for example 128) | Finer, more scales to store. What 4-bit LLM formats use |
| Per-vector, multi-level (VS-Quant) | Vector, with a coarse float scale above it | 4-bit values with a 4-bit scale per 16 elements = **4.25 effective bits** |
| Shared micro-exponent (MX) | Small block, hardware-supported | Blackwell's "micro-tensor scaling"; FP4 tensor cores give 2x the throughput of FP8 |

Multi-level scaling generalises this: `r = (q - z) * s_l0 * s_l1 * ...`, cheap integer scales at fine granularity, expensive float scales at coarse granularity.

### Clipping: what range to quantize?

Activations change with every input, so their range must be estimated:

- **During training:** exponential moving average of observed ranges.
- **Calibration batches:** run a few samples through the float model, then pick the clip value that minimises mean-squared error (for a Laplace distribution the optimum is about 2.83b, 3.89b, 5.03b for 2, 3, 4 bits) or minimises the KL divergence between float and quantized distributions (TensorRT's method).

Clipping trades a few large errors on outliers for finer resolution everywhere else.

### Rounding: up or down?

Rounding each weight to nearest is not optimal, because weights interact. **AdaRound** (Nagel et al. 2020) learns, per weight, whether to round up or down so the layer's *output* is reconstructed best. Weights only, short tuning, still close to post-training.

Results quoted (INT8): per-channel weights with KL-calibrated activations lost 0.08 to 0.45% on GoogleNet and ResNets. MobileNetV1 per-tensor collapsed (-11.8%), which per-channel fixes.

## 2. Quantization-Aware Training (QAT)

For 4 bits and below, fine-tune with quantization in the loop:

- Keep a full-precision copy of the weights; the forward pass uses quantized weights and activations ("fake quantization").
- **Straight-through estimator (STE):** rounding has zero gradient almost everywhere, so pass the gradient through as if quantization were the identity.
- MobileNetV1 INT8: 70.9% float, 0.1% with per-tensor PTQ, 59.1% with per-channel PTQ, 70.0 to 70.7% with QAT.

## 3. Binary and Ternary

- **Binary weights** (+1/-1) turn multiplies into additions; with binary activations too, a dot product becomes XNOR plus popcount (XNOR-Net). Large accuracy loss on ImageNet.
- **Ternary** (+1, 0, -1): TWN uses a threshold of 0.7 x mean |w|; TTQ learns separate positive and negative scales.

## 4. Mixed Precision

Different layers tolerate different bit widths. Searching per-layer bit widths is a huge space (64 choices per layer for 1 to 8 bits on weights and activations). **HAQ** automates it with the hardware in the loop and beats uniform quantization at the same latency, energy or size. Edge and cloud hardware end up with different policies.

## 5. Where AWQ Fits (from my run, not the lecture)

AWQ is group-wise 4-bit weight-only quantization (group size 128 in Qwen's checkpoint) with activation-aware per-channel scaling: it protects the weight channels that see large activations. In vLLM it runs on the Marlin kernel, which dequantizes INT4 to 16 bits inside the matmul. That is "W4A16": 4-bit storage, 16-bit compute.

## Key Points to Retain

- PTQ chooses granularity, clipping and rounding. Per-channel fixes outlier channels; group quantization is what makes 4-bit LLM weights work.
- Effective bits include the scales: 4-bit values with a scale every 16 elements cost 4.25 bits.
- Calibrate activations on real samples. Clip to minimise MSE or KL, not to the absolute max.
- QAT plus the STE recovers accuracy at low bits, at the cost of training.
- Weight-only formats (W4A16) save memory and bandwidth; weight-and-activation formats (W8A8 FP8) also speed up compute. That distinction explained most of my L40S results.
