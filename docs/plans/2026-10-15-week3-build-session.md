# Week 3 Build Session: Measurement Dashboard

**Date:** Thu 2026-10-15, one early session (~3.5-4 hours with breaks).
**Covers:** Week 3 Tue, Wed, Thu builds.
**Setup:** Qwen3-8B on vLLM, Modal A10 (from week 2). Dashboard config goes in the `inference-engineering` repo; notes go here.

## Status: done 2026-10-18

Live run complete: trickle + llm-slo-bench ramp (300/300, exit 0). Results in `notes/week3-build-log.md`; post draft in `socials.md`; llm-slo-bench README "Measured results" filled; week 3 ticked. Nothing committed yet. Follow-ups and commit plan: `docs/plans/2026-10-18-tonight-todo.md`. A bonus run through sre-inference-gateway is written up in that repo (`docs/measurements/2026-10-18-vllm-modal/`).

## Prep notes (2026-10-18, on the Pi)

Block 0 is done, uncommitted, in `~/REPOS/inference-engineering/observability/` (see its README for the run steps):

- Compose stack pinned, Grafana on host port 3001 (3000 is the buzz relay), datasource and dashboard provisioned. YAML/JSON parse checked; `docker compose up` not run (the agent-host guard reserves it for the owner).
- Dashboard edited: queue and prefill/decode panels fixed (they plotted `rate(_sum)` alone), requests/s panel added in place of "Max Generation Token". Panel-definition table in the README, which is Block 4's mapping done ahead of time; check it against live data.
- `trickle.sh` and `slo-bench-modal.yaml` written. The config passes llm-slo-bench validation.
- llm-slo-bench cloned to `~/REPOS/llm-slo-bench` and built for arm64; `make demo-html` passes (needs `GOCACHE=$TMPDIR/gocache` inside the agent sandbox).
- `deploy/modal/vllm_qwen3_8b.py` already has `max_containers=1` and `unauthenticated=True`, and serves the model as `qwen3-8b` with no reasoning parser.

Remaining blocker: the Modal CLI isn't installed or logged in on the Pi (`uv tool install modal && modal setup`, owner only). Then Blocks 1-3 live, Block 4 check, Block 5 write-up.

## Goal

By the end, you have:

1. A live Grafana dashboard showing TTFT, ITL, throughput, and queue depth for your Modal vLLM endpoint.
2. Panels that visibly move under load, driven by the trickle script and an [llm-slo-bench](https://github.com/crypticseeds/llm-slo-bench) ramp.
3. Every panel mapped to a written metric definition, with undefined panels deleted.
4. Client-side TTFT from llm-slo-bench compared with server-side TTFT from the dashboard.
5. A short write-up and a week 3 social post draft.

## Load Sources

| Tool | Role | When |
|---|---|---|
| `scripts/trickle.sh` (curl loop) | Fastest way to make the panels move after the dashboard is imported | Block 3, first |
| `llm-slo-bench ramp` | Structured open-loop ramp against the real endpoint, with client-side TTFT and chunk ITL | Block 3, second |

GuideLLM, `vllm bench serve`, and genai-perf stay in week 5.

## Gotchas (Read First)

### Modal

- **Prometheus can only scrape over HTTPS.** Prometheus runs on your laptop in Docker, so it scrapes the public `https://<workspace>--<app>.modal.run/metrics` URL, not `localhost:8000`. You need `scheme: https` in the scrape config.
- **Pin to one container.** If Modal starts more than one replica, each scrape hits a random container, so counters jump around and gauges look wrong. Set `max_containers=1` for this session.
- **Scrapes keep the GPU awake and billing.** A scrape every 5 seconds counts as traffic, so the container never scales to zero. **Stop Prometheus (`docker compose down`) as soon as you finish**, or you keep paying for the A10.
- **Counters reset on restart.** A cold start or redeploy resets all counters to zero. `rate()` handles this, but raw counter graphs will show a drop.
- **Auth.** If the endpoint needs an API key or Modal proxy auth, Prometheus needs it too. In Prometheus, use `authorization` / `credentials_file` in the scrape config. Keep the key in Doppler, never in a committed file.

### llm-slo-bench against Modal vLLM

- **The Modal URL needs an API key setting.** The config validator rejects a non-loopback `base_url` unless `api_key_env` is set. Point it at the env var holding your endpoint key and run it through `doppler run --only-secrets <NAME> -- ./bin/llm-slo-bench ...`. If vLLM was started without `--api-key`, any non-empty value works, because vLLM ignores the header.
- **Qwen3 thinking will distort TTFT.** llm-slo-bench counts TTFT from the first non-empty `delta.content`. If vLLM runs a reasoning parser, the thinking tokens arrive as `reasoning_content`, so the measured "TTFT" becomes "time until thinking ends". With a small `max_completion_tokens`, a request may never reach content, and the tool records it as failed. The config schema has no `chat_template_kwargs` field, so put Qwen3's `/no_think` switch in the prompt, then check with `probe` that content arrives quickly.
- **The tool drops arrivals instead of queueing them.** When `max_in_flight` is full, new arrivals are dropped and counted. For this week, keep the rates low enough that `dropped_rate` stays at 0. You're only making panels move, not finding capacity (that's week 5).

## Plan

### Block 0: Prep without the GPU (40 min, before starting Modal)

- Copy `docker-compose.yaml`, `prometheus.yaml`, and `grafana.json` from [vllm/examples/observability/prometheus_grafana](https://github.com/vllm-project/vllm/tree/main/examples/observability/prometheus_grafana) into `inference-engineering/observability/`. (The docs page linked in the README now returns a 404; the GitHub folder is the current source.)
- Edit the `prometheus.yaml` scrape job: target your Modal host, `scheme: https`, `scrape_interval: 5s`.
- Add `max_containers=1` to the Modal app.
- Write `scripts/trickle.sh`: a loop that sends a chat completion every second or so, with a few requests in parallel.
- In llm-slo-bench, run `make build`, then `make demo-html` against the mock to confirm the binary works.
- Write `observability/slo-bench-modal.yaml`, starting from `examples/quickstart.yaml`:
  - `target.base_url: https://<host>/v1`, `model: Qwen/Qwen3-8B` (match the served model name), `api_key_env: <NAME>`
  - `request.prompt`: a short prompt ending in `/no_think`; `max_completion_tokens: 128`; `timeout: 60s`; `stream_idle_timeout: 10s`
  - `load`: `max_in_flight: 8`, with stages of about 2 min each at 0.5, 1, then 2 RPS
  - `safety`: `max_duration` at or above the sum of the stages, and `max_requests` around 500
  - `slo`: a loose `p99_ttft_ms` (for example 2000) plus `max_error_rate: 0.01`. This is a sanity gate, not a real SLO yet.
- Pin the versions you'll write down later: vLLM, Grafana, Prometheus, and the llm-slo-bench commit.

**Done when:** `docker compose config` validates, the Modal app diff shows `max_containers=1`, the trickle script exists, `make demo-html` exits 0, and the Modal config file exists.

### Block 1: Endpoint up, metrics reachable (20 min)

- Deploy or serve the Modal app and wait for the model to load.
- `curl https://<host>/metrics | grep -E 'vllm:(num_requests|kv_cache_usage|time_to_first_token)'`
- `doppler run --only-secrets <NAME> -- ./bin/llm-slo-bench probe --endpoint https://<host>/v1 --model <model> --api-key-env <NAME> --prompt "Say hi /no_think"` (check the exact flags with `probe --help`)

**Done when:** the curl returns `vllm:num_requests_running`, `vllm:num_requests_waiting`, `vllm:kv_cache_usage_perc`, and TTFT histogram buckets, and `probe` returns non-empty content with a semantic TTFT in a sensible range (no thinking delay).

### Block 2: Prometheus + Grafana live (40 min)

- `docker compose up -d`
- In Prometheus at `localhost:9090/targets`, confirm the job is **UP**. If it's down, fix the scrape config now: scheme, host, path, or auth.
- In Grafana at `localhost:3000` (admin/admin): add the Prometheus data source at `http://prometheus:9090`, then import `grafana.json`.

**Done when:** Prometheus shows the target as UP and the imported dashboard loads without data source errors.

### Block 3: Panels move under load (40 min)

1. Run the trickle script for a couple of minutes. Check that the TTFT, ITL, tokens/s, and running-requests panels respond.
2. Run the ramp. Write down the wall-clock start and end times; you'll need them in Block 4.
   ```bash
   doppler run --only-secrets <NAME> -- ./bin/llm-slo-bench ramp \
     --config observability/slo-bench-modal.yaml \
     --out artifacts/week3-ramp.json --html artifacts/week3-ramp.html \
     --raw-jsonl artifacts/week3-ramp.jsonl
   ```
3. Watch the dashboard step up with each ramp stage. Note any panel that stays empty, such as `num_requests_swapped`, which V1 removed.
4. Take a dashboard screenshot during the highest stage, for the post.

**Done when:** the panels change with each ramp stage, the ramp exits 0 or 1 (not 2 or 3), `artifacts/` has the JSON and HTML report, and you have a screenshot.

### Block 4: Map metrics to panels + compare views (50 min, GPU can be off)

**Stop the Modal app and `docker compose down` first.** The rest of the session doesn't need the GPU. Prometheus data stays in its volume; bring Prometheus and Grafana back up without Modal if you want to query the ramp window again.

- For every dashboard panel, write down its PromQL query and which metric definition it shows (TTFT, ITL, TPOT, and so on, from `notes/week3-takeaways.md`). Delete panels you can't define.
- Make sure the four deliverable panels exist: TTFT p50/p95/p99, ITL p50/p95/p99, output tokens/s plus requests/s, and queue depth next to running requests. Add KV cache usage.
- Compare llm-slo-bench's p99 TTFT and chunk ITL with the dashboard's p99 TTFT and ITL over the ramp window. Write down the gap and what explains it:
  - The client-side numbers include network + TLS to Modal; the server-side ones start at tokenization.
  - Chunk ITL is per SSE event, while vLLM's ITL is per streamed output. They should be close with one token per chunk.
  - A p99 over a few hundred requests is rough; note the sample count.
- Export the edited dashboard JSON into the repo.

**Done when:** every panel left on the dashboard has a written definition, all four deliverable panels exist, and the client-vs-server gap is written down with its reason.

### Block 5: Write-up (45 min)

- `notes/week3-build-log.md` in this repo: what you set up, the versions used, the scrape-config fixes, the panel-to-definition table, which panels you deleted and why, the client-vs-server comparison, and what surprised you.
- `socials.md`: a week 3 post draft (LinkedIn and X) built around the dashboard screenshot. Follow the existing rule: post findings, not activity. The client-vs-server TTFT gap is a good finding to lead with.
- Tick week 3 in `progress.md` (Tue to Thu builds, plus the readings you've done).
- llm-slo-bench `README.md`: replace the whole "Measured results" section (including the coordinator comment and the TBD banner) with the draft below, filled in from `artifacts/week3-ramp.json`. Every `<...>` must come from the live run's artifacts or the pinned versions; leave nothing estimated.

**Done when:** the build log and post draft exist, week 3 is ticked, and every number in the llm-slo-bench "Measured results" table comes from the live run's artifacts.

**Decision (2026-10-14):** the README reports a direct-to-vLLM run. Gateway and failover results are not part of this run.

#### Draft "Measured results" section for llm-slo-bench

```markdown
## Measured results

One live `ramp` run directly against a vLLM OpenAI-compatible endpoint, with no gateway in between. All numbers are client-side, measured from the load generator (see Limitations).

**Setup:** vLLM `<version>` serving `Qwen/Qwen3-8B` (`<dtype>`) on one NVIDIA A10 (Modal, `max_containers=1`). Load generated from `<location, e.g. a laptop in ...>` over the public internet with llm-slo-bench `<commit>`. Prompt ends in `/no_think` so semantic TTFT is not inflated by reasoning tokens. `max_completion_tokens: 128`, `max_in_flight: 8`, stages `0.5 -> 1 -> 2` req/s, `<N>` min each. Run date `<YYYY-MM-DD>`.

| Metric | Value | Notes |
| --- | --- | --- |
| p50 / p99 semantic TTFT | `<x>` ms / `<y>` ms | `<samples>` samples; includes network and TLS to Modal |
| p50 / p99 chunk ITL | `<x>` ms / `<y>` ms | per SSE content event, not per token |
| Offered load | 0.5-2 req/s ramp | the configured test load, **not** a measured capacity limit |
| Outcomes | `<ok>`/`<scheduled>` ok, `<errors>` errors, `<dropped>` dropped | |
| Avg generation throughput | `<x>` tokens/s per request | completion tokens / (duration - TTFT) |
| SLO verdict | `<pass/fail>` (exit `<code>`) | gates: `p99_ttft_ms <= 2000`, `max_error_rate <= 0.01` |

Not measured in this run: capacity at saturation, and gateway failover behaviour. Those need a dedicated run and are not inferred from this one.
```

## Shutdown Checklist

- [ ] Modal app stopped, and the Modal dashboard shows 0 containers
- [ ] `docker compose down`
- [ ] No API key in any committed file or artifact (`git diff --staged` checked)

## If Time Runs Short

Cut in this order: drop the client-vs-server comparison (keep the ramp artifacts and do it later), then shorten Block 5 to just the build log. Never skip the shutdown checklist.
