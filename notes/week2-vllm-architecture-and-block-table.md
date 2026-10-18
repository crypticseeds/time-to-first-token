# Week 2: vLLM Architecture and the Block-Table Path

This note combines the Thursday architecture reading with the Friday source-reading task. It focuses on the normal single-GPU transformer path and intentionally leaves out advanced distributed and hybrid-cache branches.

## What You Should Understand

After reading this note, you should be able to explain:

1. How a request moves through vLLM V1.
2. How the scheduler decides which tokens run next.
3. How Key-Value (KV) cache blocks are allocated.
4. How logical token positions become physical KV-cache slots.

## 1. High-Level Request Flow

```text
OpenAI-compatible API request
          |
          v
Large Language Model (LLM) Engine
          |
          v
Engine Core
          |
          v
Scheduler ----> KV Cache Manager ----> Block Pool
          |
          v
Scheduler Output containing block identifiers
          |
          v
Graphics Processing Unit (GPU) Model Runner
          |
          v
Worker-side block table and attention kernel
```

### LLM Engine

The LLM Engine handles user-facing work such as tokenization, request creation, output processing, and streaming generated text.

### Engine Core

The Engine Core repeatedly coordinates an engine step:

```text
Schedule work -> execute the model -> process output -> repeat
```

### Scheduler

The scheduler answers: **which requests run now, and how many tokens may each process?**

### KV Cache Manager

The KV Cache Manager answers: **which cache blocks belong to each request, and are enough blocks available?**

### Model Runner

The model runner prepares the selected batch, transfers block-table data to the GPU, executes the model, and returns sampled tokens.

## 2. vLLM V1 Scheduling

The main source is [`vllm/v1/core/sched/scheduler.py`](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/sched/scheduler.py).

The scheduler maintains requests that are waiting and running. At each step, it works within two important limits:

- Maximum number of requests
- Maximum number of scheduled tokens

Current vLLM V1 does not need separate prefill and decode schedulers. For each request, it compares the number of known tokens with the number already computed.

```text
tokens requiring work = known tokens - computed tokens
```

This single model supports:

- Prefill of a new prompt
- Chunked prefill of a long prompt
- Decode of the next output token
- Prefix-cache reuse
- Speculative decoding

After deciding how many tokens a request may process, the scheduler calls:

```python
KVCacheManager.allocate_slots(...)
```

If allocation fails because there are not enough blocks, the scheduler may preempt a lower-priority request. In V1, preemption usually means freeing that request's KV blocks and recomputing its state when it resumes.

## 3. KV Cache Allocation

The main source is [`vllm/v1/core/kv_cache_manager.py`](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/kv_cache_manager.py).

The KV cache stores attention keys and values from tokens already processed. Without it, decode would repeatedly calculate old attention state.

`KVCacheManager.allocate_slots()` performs the main allocation flow:

1. Count the tokens that need cache slots.
2. Account for blocks already owned by the request.
3. Account for reusable prefix-cache blocks, when enabled.
4. Calculate how many additional blocks are required.
5. Check whether enough free blocks exist.
6. Allocate blocks or return `None` if the request cannot fit.
7. Associate the resulting blocks with the request.

The manager deals in fixed-size blocks rather than requiring one contiguous memory region per request. This is the memory-management idea behind PagedAttention.

## 4. Block Pool

The main source is [`vllm/v1/core/block_pool.py`](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/block_pool.py).

The Block Pool owns the available physical KV-cache blocks. Each block has an integer identifier.

Important responsibilities include:

- `get_new_blocks()`: remove free blocks from the queue and assign them
- `free_blocks()`: return unused blocks to the queue
- `touch()`: increase a reused block's reference count
- `get_cached_block()`: find a reusable block from its prefix hash

When prefix caching is enabled, a free block may still hold valid cached data. It remains available for reuse until the allocator needs that physical block for different data. Allocation then evicts the old cached association.

### Logical and Physical Blocks

A request thinks in sequence order:

```text
logical blocks: 0, 1, 2, 3
```

The physical GPU blocks do not need to be adjacent:

```text
physical IDs:   42, 7, 105, 19
```

The request's block table stores that mapping:

```text
logical block 0 -> physical block 42
logical block 1 -> physical block 7
logical block 2 -> physical block 105
logical block 3 -> physical block 19
```

This removes the need to reserve one large contiguous KV-cache region for every request.

## 5. Sending Block Identifiers to the Worker

The interface is defined in [`vllm/v1/core/sched/output.py`](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/sched/output.py).

For a new request, `NewRequestData.block_ids` contains its assigned physical block identifiers. For a request already known to the worker, `CachedRequestData.new_block_ids` sends only newly allocated identifiers.

This avoids retransmitting the full request state during every engine step.

## 6. Building the Worker-Side Block Table

The model runner source is [`vllm/v1/worker/gpu/model_runner.py`](https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/gpu/model_runner.py).

For a new request, `add_requests()` adds its block identifiers to a block-table row. For an existing request, `update_requests()` appends newly allocated identifiers.

Conceptually, the worker holds a two-dimensional table:

```text
                 logical block index
request row       0      1      2      3
request A        42      7    105     19
request B        13     88     31      -
```

The table is copied to the GPU before model execution so the attention kernel can locate each request's KV data.

## 7. From Token Position to Physical Slot

The focused source is [`vllm/v1/worker/block_table.py`](https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/block_table.py), especially `BlockTable.compute_slot_mapping()`.

For the normal case, the mapping is:

```text
logical block index = token position // block size
offset inside block = token position % block size
physical block ID   = block table[request][logical block index]
physical slot       = physical block ID x block size + offset
```

### Example

Assume:

- Block size: 16 tokens
- Token position: 35
- Request block table: `[42, 7, 105]`

Then:

```text
logical block index = 35 // 16 = 2
offset inside block = 35 % 16 = 3
physical block ID   = block_table[2] = 105
physical slot       = 105 x 16 + 3 = 1683
```

The attention operation can now read or write the KV data for token position 35 at physical slot 1683.

## 8. Complete Normal Path

```text
1. Scheduler decides that request A may process more tokens.
2. KVCacheManager.allocate_slots() calculates required blocks.
3. BlockPool supplies physical block IDs from the free queue.
4. Scheduler stores those IDs in SchedulerOutput.
5. GPUModelRunner adds the IDs to request A's block-table row.
6. BlockTable.compute_slot_mapping() combines token positions with the row.
7. The attention kernel uses the resulting physical slot IDs.
8. When the request finishes or is preempted, its blocks return to the pool.
```

## 9. Prefix Caching Extension

Prefix caching adds lookup before allocation:

1. Tokens are grouped into full blocks.
2. A block hash includes its tokens and preceding prefix.
3. The KV Cache Manager checks whether matching blocks already exist.
4. Matching blocks are attached to the new request by increasing their reference counts.
5. Only the unmatched suffix receives new blocks.

The physical block can therefore appear in more than one request's block table when those requests share the same cached prefix.

## 10. What to Ignore on the First Read

Current vLLM supports many advanced cases. They make the source look harder than the core idea.

Ignore these initially:

- Multiple KV-cache groups
- Hybrid attention and state-space models
- Decode or prefill context parallelism
- External KV-cache connectors
- Copy-on-write partial blocks
- Speculative lookahead slots
- Kernel block sizes different from allocation block sizes

Return to them only after the normal mapping is clear.

## Three-Sentence Friday Deliverable

The scheduler asks the Key-Value (KV) Cache Manager to reserve fixed-size blocks for the tokens selected in the current engine step. The allocated physical block identifiers are sent through the scheduler output and stored in the worker's block-table row for that request. During execution, each logical token position selects a physical block from that row plus an offset within the block, producing the physical KV-cache slot used by the attention kernel.

## Revision Questions

1. What decision belongs to the scheduler rather than the KV Cache Manager?
2. Why can a request use non-contiguous physical blocks?
3. What does a block-table row contain?
4. How is a token position converted into a physical KV-cache slot?
5. What happens when allocation cannot find enough free blocks?

## Key Points

- The scheduler selects work; the KV Cache Manager selects storage.
- PagedAttention uses fixed-size, non-contiguous physical cache blocks.
- The block table maps each request's logical block order to physical block identifiers.
- Scheduler output carries block identifiers from the central scheduler to workers.
- Slot mapping combines a physical block identifier with the token's offset inside that block.
- Prefix caching reuses matching physical blocks and allocates only the unmatched suffix.
