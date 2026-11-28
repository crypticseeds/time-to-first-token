# Week 6 Mon: MIT 6.5940 Lecture 5, Quantization Part I

Source: [MIT 6.5940 Lecture 5: Quantization Part I](https://www.youtube.com/watch?v=TSc_BibWRhM) (Song Han). Notes follow the Fall 2024 slides on the [course page](https://hanlab.mit.edu/courses/2024-fall-65940).

## Main Idea

Quantization stores and computes with fewer bits. Fewer bits means less memory, less data movement and cheaper arithmetic. The lecture builds from how numbers are represented to how a whole layer runs in integers.

## 1. Why Fewer Bits

Energy per operation at 45 nm, 0.9 V (Horowitz, ISSCC 2014):

| Operation | Energy (pJ) |
|---|---:|
| 8-bit int add | 0.03 |
| 32-bit int add | 0.1 |
| 16-bit float add | 0.4 |
| 32-bit float add | 0.9 |
| 8-bit int multiply | 0.2 |
| 32-bit int multiply | 3.1 |
| 16-bit float multiply | 1.1 |
| 32-bit float multiply | 3.7 |

An 8-bit integer add is 30 times cheaper than a 32-bit float add, and an 8-bit multiply 16 times cheaper than a 32-bit float multiply. For LLM decode the bigger win is memory traffic: fewer bytes per weight means fewer bytes to read per token.

## 2. Number Formats

Exponent bits set **range**, fraction (mantissa) bits set **precision**.

| Format | Sign / exponent / fraction | Note |
|---|---|---|
| FP32 | 1 / 8 / 23 | IEEE single |
| FP16 | 1 / 5 / 10 | More precision, small range |
| BF16 | 1 / 8 / 7 | FP32's range, less precision. The usual LLM weight format |
| FP8 E4M3 | 1 / 4 / 3 | No infinity; largest normal value **448**. Used for weights and activations |
| FP8 E5M2 | 1 / 5 / 2 | Has infinity and NaN; largest normal **57,344**. Used for gradients |
| INT4 | 4 bits | -8 to 7 |
| FP4 E2M1 | 1 / 2 / 1 | Values 0, 0.5, 1, 1.5, 2, 3, 4, 6 |

Floating-point details from the slides: exponent bias (127 for FP32), normal numbers `(1 + fraction) * 2^(exp - bias)`, and subnormals when the exponent is all zeros, which extend the range toward zero.

## 3. Three Kinds of Quantization

| Kind | Stored | Computed in |
|---|---|---|
| K-means (weight sharing) | Integer cluster indices + a float codebook | Floating point (decode the codebook) |
| Linear | Integers + scale and zero point | Integers |
| Binary / ternary | 1 or 2 bits | Bit operations (Lecture 6) |

### K-means quantization (Deep Compression, Han et al. 2016)

Cluster the weights, store each weight as a small index into a codebook of centroids, then fine-tune the centroids. Combined with pruning and Huffman coding it compressed AlexNet 35x (240 MB to 6.9 MB) and VGGNet 49x with no accuracy loss. It saves storage but the math is still floating point.

## 4. Linear Quantization

An affine map between integers and real numbers:

```text
r = S (q - Z)

S = (r_max - r_min) / (q_max - q_min)        scale (float)
Z = round(q_min - r_min / S)                 zero point (integer)
```

- **Symmetric:** Z = 0 and the range is `[-|r|max, |r|max]`. With N bits, `q_min = -2^(N-1)`, `q_max = 2^(N-1) - 1`, and in full-range mode `S = |r|max / 2^(N-1)`.
- **Asymmetric:** a zero point shifts the range, which suits values that are not centred on zero (activations after ReLU).

### A quantized matrix multiply

With weight zero point `Z_W = 0`:

```text
q_Y = (S_W S_X / S_Y) (q_W q_X - Z_X q_W) + Z_Y
       \___ rescale __/ \__ N-bit int mult, 32-bit int add __/
```

- The product of two N-bit integers is accumulated in 32-bit integers.
- `Z_X q_W` depends only on the weights, so it is precomputed.
- The float factor `S_W S_X / S_Y` is usually a fixed-point multiply plus a bit shift.
- A bias is folded in as a 32-bit integer with `S_b = S_W S_X` and `Z_b = 0`. Convolution works the same way.

Result reported (Jacob et al. 2018): ResNet-50 76.4% (float) vs 74.9% (INT8), Inception-V3 78.4% vs 75.4%.

## Key Points to Retain

- Range comes from exponent bits, precision from fraction bits. BF16 keeps FP32's range; FP8 E4M3 tops out at 448, so scale factors are essential.
- K-means quantization saves storage only; linear quantization saves compute too, because the matmul runs in integers.
- `r = S(q - Z)`. Symmetric (Z = 0) for weights, asymmetric often for activations.
- Integer matmul accumulates in 32 bits, then rescales to N bits.
- In my Week 6 run, only the FP8 checkpoint computes in 8 bits. AWQ stores 4-bit weights but computes in 16 bits, which is why it helps memory-bound decode and not compute-bound large batches.
