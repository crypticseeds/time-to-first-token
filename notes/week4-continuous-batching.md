# Week 4 Thu: Continuous Batching

Source: [How continuous batching enables 23x throughput in LLM inference while reducing p50 latency](https://www.anyscale.com/blog/continuous-batching-llm-inference) (Anyscale, 2023-06-22).

## Main Idea

LLM inference is **memory-IO bound**: each decode step reads all the weights to produce one token per sequence. Batching amortizes that read over many sequences, so throughput depends on how big a batch fits in GPU memory. How you batch matters as much as how big:

- **Static batching** keeps the batch fixed until every sequence finishes. Short answers sit idle waiting for the longest one.
- **Continuous batching** (iteration-level scheduling) decides the batch **every iteration**. A finished sequence leaves and a waiting one takes its slot at the next step.

## 1. Why Static Batching Wastes the GPU

```text
static:      [A A A A A A A A]   B finished at step 3, its slot idles until A finishes
             [B B B . . . . .]
             [C C C C C . . .]

continuous:  [A A A A A A A A]
             [B B B D D D D D]   D joins as soon as B leaves
             [C C C C C E E E]
```

The waste grows with **variance in output length**. When outputs are similar in length, the two approaches perform about the same.

Background numbers from the article: prefill of the prompt takes about as long as one decode step, and a 13B model holds nearly **1 MB of state per token**, so memory decides batch size.

## 2. Where PagedAttention Fits

vLLM's PagedAttention stores the KV cache in fixed-size blocks allocated just in time, limiting waste to under **4%** (only the last block). Less waste means a bigger batch, and continuous batching keeps that bigger batch full. The two together give the headline result.

## 3. The Benchmark

| Item | Value |
|---|---|
| Hardware and model | 1 NVIDIA A100 40 GB, OPT-13B, no tensor parallelism |
| Throughput test | 1,000 requests, 512 input tokens each, output length from an exponential distribution (mean 128), capped at 32, 128, 512 or 1,536 |
| Latency test | 100 requests, input 1 to 512 tokens, Poisson arrivals at a fixed QPS |
| Static batching | HF Pipelines, NVIDIA FasterTransformer |
| Continuous batching | HF text-generation-inference, Ray Serve, vLLM |

### Throughput (tokens/s) by max generation length

| System | 32 | 128 | 512 | 1,536 |
|---|---:|---:|---:|---:|
| HF Pipelines (static) | 104 | 271 | 181 | 81 |
| FasterTransformer (static) | 106 | 418 | 729 | 317 |
| TGI (continuous) | 105 | 410 | 699 | 739 |
| Ray Serve (continuous) | 104 | 412 | 694 | 744 |
| vLLM (continuous + paged) | 257 | 981 | 2,197 | 2,307 |

At length 1,536, vLLM vs naive static batching is the **23x**. At length 32, the continuous systems without paging are no better than static: low variance, nothing to gain.

### Median latency (s)

| System | QPS 1 | QPS 4 |
|---|---:|---:|
| HF Pipelines | 7.22 | 18.00 |
| FasterTransformer | 2.12 | 14.80 |
| TGI | 1.15 | 4.39 |
| Ray Serve | 1.14 | 4.41 |
| vLLM | 0.96 | 1.25 |

vLLM saturated around **QPS 8** at about 1,900 tokens/s.

## 4. The Caveat

The benefit is conditional on **heavy-tailed output lengths**. The article chose an exponential distribution on purpose. With uniform lengths, static and continuous batching converge. A benchmark with fixed output lengths (like my week 5 runs, 16 tokens every request) will not show this gain.

## Key Points to Retain

- Continuous batching = schedule per iteration, not per request. Finished sequences leave, new ones join between steps.
- It improves both throughput and latency at every percentile under moderate load.
- The size of the win is set by output length variance and by how big a batch memory allows.
- 23x is vLLM (paging + continuous batching) vs naive static batching at the longest, most variable outputs. Continuous batching alone gave about 9x there (744 vs 81).
- Read the [Orca note](week4-orca.md) for where the idea came from.
