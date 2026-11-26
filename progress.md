---
name: Progress tracker
about: Track your way through the roadmap. Open this in your own fork.
title: 'Roadmap progress: [your name or handle]'
labels: progress
---

Tick sessions as you finish them. GitHub saves checkbox state to your account, so this works across every device you sign in from.

Started on:

### Week 1: The mental model
- [x] Mon. Making Deep Learning Go Brrrr, first half
- [x] Tue. Making Deep Learning Go Brrrr, second half
- [x] Wed. CS336 Lecture 5, GPUs, first 30 min
- [x] Thu. CS336 Lecture 5, second 30 min
- [x] Fri. Hand-derive arithmetic intensity for your model

### Week 2: vLLM
- [x] Mon. PagedAttention announcement and paper
- [x] Tue. Serve a 7-8B model behind vLLM
- [x] Wed. vLLM V1 architecture
- [x] Thu. Inside vLLM, scheduler and block manager
- [x] Fri. Follow the block table code path

### Week 3: Measurement
- [x] Mon. vLLM metrics design doc
- [x] Tue. Stand up Prometheus and Grafana
- [x] Wed. Import the dashboard, confirm live panels
- [x] Thu. Map every metric to a panel
- [x] Fri. Modal benchmarking methodology

### Week 4: SGLang and batching
- [x] Mon. RadixAttention
- [x] Tue. Deploy SGLang with the same model
- [x] Wed. SGLang talk and paper
- [x] Thu. Continuous batching and Orca
- [x] Fri. Sarathi-Serve, chunked prefill
- [ ] Buffer. Measure shared-prefix TTFT on SGLang and vLLM (deferred until after the Week 5 concurrent session)

[Week 4 reading summary and build recap](notes/week4-takeaways.md)

### Week 5: Load testing
- [x] Mon. GuideLLM intro and install
- [x] Tue. First GuideLLM sweep
- [x] Wed. vllm bench serve cross-check
- [x] Thu. Concurrency sweep (GuideLLM concurrent profile instead of genai-perf, run from a Modal client)
- [x] Fri. Push past 1000 concurrent, and read vLLM tuning strategies
- [ ] Buffer. Pre-publish benchmark checklist

[Week 5 load-testing summary and results](notes/week5-takeaways.md)

### Week 6: Quantization
- [ ] Mon. MIT 6.5940 Lecture 5, first half
- [ ] Tue. Lecture 5 and 6, AWQ, GPTQ, SmoothQuant
- [ ] Wed. Quantization survey, map methods to engines
- [ ] Thu. Serve AWQ or GPTQ and FP8 variants
- [ ] Fri. Quality proxy across FP16, FP8, INT4

### Week 7: Speculative decoding and KV eviction
- [ ] Mon. Speculative decoding docs and benchmark post
- [ ] Tue. GPU MODE Lecture 22, first half
- [ ] Wed. Benchmark speculative decoding at low and high QPS
- [ ] Thu. StreamingLLM
- [ ] Fri. Long-context eviction, measure memory and TTFT

### Week 8: Disaggregation and Kubernetes
- [ ] Mon. DistServe talk and paper
- [ ] Tue. Splitwise and vLLM disaggregated prefilling
- [ ] Wed. Mooncake and SGLang PD disaggregation
- [ ] Thu. Deploy with the production stack
- [ ] Fri. Autoscale on queue depth, not CPU

### Week 9: Unit economics and the router
- [ ] Mon. Inference economics from first principles
- [ ] Tue. Re-derive cost per million tokens on your own GPU
- [ ] Wed. Character.AI inference posts
- [ ] Thu. Build the router
- [ ] Fri. Token budgeting middleware

### Week 10: Publish
- [ ] Mon. Draft the benchmark writeup
- [ ] Tue. Add labeled variants, publish
- [ ] Wed. Set up the weekly reading habit
- [ ] Thu. First habit session
- [ ] Fri. Optional edge sampler

### Pre-publish checklist
- [ ] Swept the request rate
- [ ] Reported exact input and output token lengths
- [ ] Reported p50, p95, p99
- [ ] Kept TTFT and inter-token latency separate
- [ ] Did not measure at the preemption point
- [ ] Used realistic, not uniform, length distributions
- [ ] Accounted for tokenizer differences
- [ ] Pinned versions and published exact commands

---

If you finish, link your benchmark writeup here. Open a PR to add it to the README.
