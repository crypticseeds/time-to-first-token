# Week 4 Thu: Orca

Source: [Orca: A Distributed Serving System for Transformer-Based Generative Models](https://www.usenix.org/conference/osdi22/presentation/yu) (OSDI 2022, [PDF](https://www.usenix.org/system/files/osdi22-yu.pdf)).

## Main Idea

Serving systems of the time (Triton with FasterTransformer) scheduled **whole requests**. The engine took a batch and returned only when every request in it was done. For autoregressive generation, where each request needs a different number of iterations, that causes two problems:

1. **Early-finished requests** wait for the slowest request in the batch, and the engine keeps computing on them.
2. **Late-joining requests** wait until the whole current batch finishes, inflating queue time.

Orca's answer is **iteration-level scheduling** plus **selective batching** to make it possible. This is the paper behind what everyone now calls continuous batching.

## 1. Iteration-Level Scheduling

```text
loop:
  scheduler picks requests from the pool
  engine runs ONE iteration on that batch
  scheduler gets one new token per request
  finished requests return to the client immediately
  new arrivals are eligible at the next iteration
```

A new request waits at most one iteration to start, and nothing waits for a slower neighbour to finish.

## 2. Selective Batching

The obstacle: in one iteration, requests have processed different numbers of tokens (some are in prefill with different prompt lengths, some are decoding at different positions). Their tensors have different shapes.

| Operation | Batched? | How |
|---|---|---|
| Linear, LayerNorm, Add, GeLU (non-attention) | Yes | Flatten all tokens into one `[sum of lengths, hidden]` tensor, no batch dimension |
| Attention | No | `Split` per request, run attention separately, `Merge` back |

Why skipping attention costs little: **attention has no weights**. Batching pays off by reusing loaded parameters across requests, and attention has none to reuse. (Later systems solved the shape problem inside the kernel instead, for example PagedAttention.)

## 3. The Scheduler

- **FCFS at iteration level:** a request that arrived earlier has run at least as many iterations as a later one.
- **`max_bs`:** maximum batch size, the throughput vs latency knob.
- **KV slot reservation:** K/V must live until a request finishes. When a request is first scheduled, Orca reserves `max_tokens` slots for it. If the reservation would exceed the pre-allocated `n_slots`, the request waits. This avoids deadlock but reserves memory for the worst case, the waste PagedAttention later removed.

## 4. Distributed Design

- **Intra-layer** (tensor) and **inter-layer** (pipeline) parallelism for models up to 341B.
- Engine master coordinates workers; each worker owns a pipeline stage and can sit on another machine.
- **Control plane on gRPC, data plane on NCCL.** Control messages (request IDs, token index, input sizes) skip the GPU, which cut CPU-GPU sync overhead.

## 5. Evaluation

| Item | Value |
|---|---|
| Hardware | Azure ND96asr A100 v4: 8x A100 40 GB per VM over NVLink, up to 4 VMs (32 GPUs), InfiniBand between VMs |
| Models | GPT-3 sizes in FP16, max sequence 2,048: 13B (1 GPU), 101B (8), 175B (16), 341B (32) |
| Workload | Synthetic Poisson arrivals, input tokens U(32, 512), max generated tokens U(1, 128) |

- **36.9x throughput at the same latency** vs FasterTransformer on GPT-3 175B: at 190 ms median normalized latency, 6.81 req/s vs 0.185 req/s.
- The engine alone, without the scheduler, beat FasterTransformer by up to **47%** on 175B, credited to the control/data plane split.

## 6. Limitations

- No real checkpoint or text: synthetic traces mean end-of-sequence timing is not realistic.
- Scheduler and engine are tightly coupled. A clean general interface between serving system and engine is left as future work.

## Key Points to Retain

- The unit of scheduling is **one iteration**, not one request.
- Selective batching works because attention has no weights to share.
- Orca's KV reservation for `max_tokens` is the over-allocation PagedAttention later fixed.
- The 36.9x is against request-level batching under varied output lengths. Like the [continuous batching article](week4-continuous-batching.md), the gain depends on length variance.
