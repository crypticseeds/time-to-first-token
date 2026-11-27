# Week 4 Takeaways: SGLang and Batching

Quick recap of the week 4 readings. Full notes:

- [RadixAttention and SGLang](week4-radixattention-sglang.md) (Mon)
- [SGLang paper and talk](week4-sglang-paper.md) (Wed)
- [Continuous batching](week4-continuous-batching.md) (Thu)
- [Orca](week4-orca.md) (Thu)
- [Sarathi-Serve and chunked prefill](week4-sarathi-serve.md) (Fri)
- [SGLang v0.4 zero-overhead scheduler](week4-sglang-v0-4-scheduler.md) (Buffer)

What I built: [Week 4 build log](week4-build-log.md).

## The One-Line Version

**A serving engine wins by not wasting work: don't recompute shared prefixes, don't leave batch slots idle, don't let one long prompt freeze every stream, and don't let the CPU keep the GPU waiting.**

## Four Kinds of Waste and Their Fixes

| Waste | Fix | Where | Wins most when |
|---|---|---|---|
| Recomputing a shared prefix | Prefix cache: RadixAttention (radix tree, LRU on leaves) | SGLang; vLLM `--enable-prefix-caching` | Shared system prompts, few-shot, multi-turn, agents |
| Finished sequences holding batch slots | Continuous batching (iteration-level scheduling) | Orca, then every modern engine | Output lengths vary a lot |
| A long prefill stalling every decode stream | Chunked prefill + stall-free scheduling under a token budget | Sarathi-Serve; vLLM `--max-num-batched-tokens` | Long prompts mixed with streaming users |
| GPU idle while the CPU schedules | Overlap scheduling one batch ahead | SGLang v0.4 (`--disable-overlap` to turn off) | Small models, short steps, large tensor parallelism |

PagedAttention (week 2) sits under all of these: it decides where KV blocks live. RadixAttention decides which of them can be reused.

## Ten Things to Remember

1. **Prefix reuse mostly helps TTFT,** because prefill is the work it skips. Production hit rates in the paper: 52 to 74%, with first-token latency 1.7x lower.
2. **RadixAttention costs almost nothing without hits** (tree overhead under 0.3%), which is why SGLang never turns it off.
3. **Cache-aware scheduling (longest prefix first) trades fairness for hit rate.** Requests with no shared prefix can starve.
4. **Across replicas, prefix caching needs cache-aware routing.** v0.4's router took the hit rate from 20% to 75%; round-robin spreads each prefix everywhere.
5. **Continuous batching = schedule per iteration.** Finished sequences leave and new ones join between steps.
6. **The batching gain depends on output length variance.** With uniform lengths, static and continuous batching converge. The 23x and 36.9x headlines both used varied lengths.
7. **Selective batching works because attention has no weights.** Batch the linear layers token-wise, run attention per request.
8. **Prefill is compute-bound, decode is memory-bound,** so a decode batch has spare compute for a bounded slice of prefill.
9. **Generation stalls** are what a prefill-first scheduler does to P99 time-between-tokens. The token budget is the knob: smaller for smooth streams, larger for throughput and TTFT. Keep it a tile multiple (257 tokens was 32% slower than 256).
10. **A fair engine comparison pins everything,** including the settings with different names in each engine: prefix caching, CUDA graphs, memory fraction, batch cap, sampling defaults.

## Numbers Worth Knowing

| Claim | Number | Conditions |
|---|---|---|
| SGLang throughput | up to 5x (blog), 6.4x (paper) | Prefix-heavy LM programs, A10G, vs vLLM 0.2.5 and Guidance |
| Continuous batching | 23x (vLLM vs naive static) | OPT-13B, 1x A100 40 GB, outputs up to 1,536 tokens, exponential lengths |
| Orca | 36.9x at equal latency | GPT-3 175B on 16 A100s vs FasterTransformer |
| Sarathi-Serve | 2.6x to 5.6x serving capacity | Under P99 TBT SLOs, Mistral-7B to Falcon-180B |
| SGLang v0.4 overlap scheduler | 1.1x vs previous, 1.3x vs baselines | Llama-3.2-3B, 4k in / 2k out |

Every headline number above comes from a workload chosen to show the technique. Read the conditions before quoting the multiplier.

## Week 4 Status

- [x] SGLang deployed on the same A10, model revision and limits as vLLM ([build log](week4-build-log.md)).
- [x] Shared-prefix TTFT comparison against vLLM, run 2026-11-27 ([results in the build log](week4-build-log.md#shared-prefix-comparison-buffer-run-2026-11-27)). A hit cut one-request TTFT from 533 to 73 ms on vLLM and from 607 to 170 ms on SGLang.

## Sources Note

- The SGLang talk is a video. These notes follow the paper, which covers the same system.
- The Orca link in the roadmap is the USENIX presentation page; the paper PDF is `usenix.org/system/files/osdi22-yu.pdf`.
