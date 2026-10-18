# To do: week 3 wrap-up (from 2026-10-18)

## Done 2026-10-18

| Repo | Branch | PR |
|---|---|---|
| inference-engineering | `feat/week3-observability` | https://github.com/crypticseeds/inference-engineering/pull/1 |
| llm-slo-bench | `docs/live-vllm-results` | https://github.com/crypticseeds/llm-slo-bench/pull/1 |
| sre-inference-gateway | `fix/forward-messages-without-null-name` | https://github.com/crypticseeds/sre-inference-gateway/pull/35 |
| sre-inference-gateway | `docs/real-vllm-baseline` (includes the mock-lane methodology, now in `docs/measurements/`) | https://github.com/crypticseeds/sre-inference-gateway/pull/36 |
| time-to-first-token (fork) | `docs/week3-measurement` | this branch's PR |

## Still to do

- [ ] **Review and merge, in this order:** inference-engineering #1 (the others link to it), llm-slo-bench #1, gateway #35, then gateway #36 (its run used #35's fix), then the fork PR.
- [ ] **Upstream roadmap PR, stale week 3 links.** Branch `fix/stale-week3-links` in worktree `~/REPOS/ttft-link-fix`, built from `upstream/main`, so it carries only this change. 7 lines across `README.md` and `index.html`:
  - vLLM metrics doc `design/v1/metrics/` -> `design/metrics/` (old one redirects)
  - vLLM `online_serving/prometheus_grafana/` -> GitHub `examples/observability/prometheus_grafana` (404)
  - NVIDIA `llm-inference-benchmarking-fundamental-concepts/` -> `llm-benchmarking-fundamental-concepts/` (404)
  Push to your fork, then PR into patchy631:
  https://github.com/patchy631/time-to-first-token/compare/main...crypticseeds:time-to-first-token:fix/stale-week3-links
- [ ] **Post week 3.** Draft at the end of `socials.md`; attach the ramp screenshot. Post after inference-engineering #1 merges (the post links to `observability/`).
- [ ] **Cleanup after merges:** `git worktree remove` for `~/REPOS/sre-inference-gateway-fix-null-name`, `~/REPOS/sre-inference-gateway-real-vllm-baseline` and `~/REPOS/ttft-link-fix`, then `git branch -d` the merged branches.

## Someday (gateway roadmap)

The gateway adds ~100 ms to the first token on its upstream hop. Steps are in the gateway README's Roadmap: prove connection reuse is the cause, then tune keep-alive / pool / HTTP/2, add a connection-reuse metric, re-measure in-region, and run a real-backend failover drill.
