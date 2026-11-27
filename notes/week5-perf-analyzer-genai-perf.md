# Week 5 Thu: perf_analyzer and GenAI-Perf

Source: [triton-inference-server/perf_analyzer](https://github.com/triton-inference-server/perf_analyzer) README.

## Main Idea

perf_analyzer is NVIDIA's CLI for measuring inference performance on Triton Inference Server. GenAI-Perf was its LLM-focused front end, the tool the roadmap named for Thursday's concurrency sweep. **GenAI-Perf is now deprecated.** The README says:

> genai-perf is being deprecated. Please migrate to [AIPerf](https://github.com/ai-dynamo/aiperf) for continued support and enhanced features.

AIPerf lives in the `ai-dynamo` GitHub organization.

## 1. Load Modes

| Mode | How it generates load |
|---|---|
| **Concurrency** | Keeps a fixed number of requests outstanding (closed loop) |
| **Request rate** | Sends requests at a fixed rate (open loop) |
| **Custom interval** | Sends requests at specified intervals |

Concurrency mode is the same idea as GuideLLM's `concurrent` profile and `vllm bench serve --max-concurrency` with an infinite rate.

## 2. Measurement Modes

| Mode | When it stops |
|---|---|
| Time windows | Measures over a fixed interval, repeats until results stabilize |
| Count windows | Measures over a fixed number of requests, repeats until stable |

"Repeat until stable" is a useful idea GuideLLM does not do by default: it guards against reporting a window that was still warming up.

## 3. Why I Did Not Use It

- GenAI-Perf is deprecated; the replacement is AIPerf.
- The NVIDIA SDK image that ships it is about **7.5 GiB**, too large for the Pi.
- GuideLLM's `concurrent` profile does the same sweep, and was already installed and pinned.

So Thursday ran GuideLLM's concurrency profile from a Modal client instead. The cost: the H100 concurrency sweep has no second-tool cross-check. Wednesday's `vllm bench serve` cross-check covers the A10 rate sweep only.

## Key Points to Retain

- Roadmap links go stale; the tool named may be deprecated by the time you get there. Check the README first.
- Concurrency (closed loop) vs request rate (open loop) is a choice every load tool offers. Know which one each number came from.
- Steady-state detection (repeat until stable) is worth copying into any custom harness.
- AIPerf is the tool to try if a second H100 cross-check is ever needed.
