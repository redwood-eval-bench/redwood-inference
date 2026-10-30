# Correctness-Latency Controller: adaptive precision & fast-path throttling

Motivation: We see a recurring tradeoff where aggressive fast-path kernel selection and low-precision variants reduce p99 latency but occasionally produce small numerical regressions on a narrow set of prompts (rare long-tail token distributions). This PR introduces the "Correctness-Latency Controller" (CLC): a runtime component that continuously balances correctness and latency targets by (1) adaptively toggling precision modes on per-model/route windows, (2) soft-throttling fast-path usage under detected correctness drift, and (3) adjusting batching windows and KV cache staleness heuristics to reduce false-positive regressions from cache evictions.

Goals: (a) keep p99 and p95 within existing SLO tiers while reducing end-to-end correctness regressions (measured by our eval harness) by >60% on canary traffic; (b) do not increase median latency by more than 6% for high-throughput routes.

What changed:
- New component runtime/clc_controller.{cc,h}: controller loop, state machine, and pluggable policies (latency-first, correctness-first, balanced). The controller consumes lightweight telemetry (per-route output-diff signal, kernel-fallback counts, kv-eviction rates, batching window occupancy) and emits decisions: precision_hint, fastpath_allowance (0..100), batching_window_ms_override, kv_staleness_grace_ms.
- Scheduler integration runtime/scheduler.cc: scheduler now queries CLC when making fast-path vs compatibility-path dispatch decisions; a soft-throttle token bucket is applied before selecting optimistic kernels.
- KV cache manager runtime/kv_cache_manager.cc: supports staleness grace window and exposes eviction surge metrics.
- Adaptive testing harness tests/clc_integration_test.py: synthetic correctness perturbations and latency SLO simulation; reproduces past regression scenarios found by ENG-8453.
- Docs and runbook docs/operational/clc.md: operational knobs, how to promote policies, rollback steps.
- Benchmarks benchmarks/clc_canary_report.md and benchmark scripts for automated canary runs.

Backwards compatibility: Controller defaults to a no-op (pass-through) policy unless enabled via feature flag service.flags.enable_clc on a route or deployment; safe-mode flag forces compatibility path and disables precision upgrades.

Tests and validation:
- Unit tests for controller state transitions and policy evaluation.
- Integration test that replays historical regression traces (from SUP-3124) and verifies controller suppresses the fast-path for offending traces.
- Canary job: 5% traffic for 48h with shadowed metrics and automatic rollback if eval-regression > 3% relative.

Benchmarks:
- Internal microbench (measured on p3.8xlarge equivalent): baseline (no CLC): median=28ms, p95=72ms, p99=130ms, tokens/s=480K. With default balanced policy: median=29.5ms (+5%), p95=74ms (+2.8%), p99=96ms (-26%); tokens/s=455K (-5%). With correctness-first: median=31ms (+10%), p99=88ms (-32%). Eval-harness (10K prompts targeted at edge cases): baseline correctness-failure-rate=8.4%, balanced policy=2.9% (-65%), correctness-first=1.6% (-81%).

Rollout plan:
- Feature flag gated.
- Start with internal dedicated clusters, then 5% canary on Hosted API for 48h.
- If canary metrics good, 50% for 24h, then full rollout.
- Automatic rollback on canary if eval regressions or latency SLO violations (+p99 > 20% absolute).

Checklist:
- [x] Unit tests
- [x] Integration replay tests
- [x] Docs and runbook
- [x] Benchmarks attached
- [x] Canary configuration in deployment manifests
- [x] Security review (no new surface)
- [x] Feature flag gated
- [x] Performance budget validated

Commits summary:
- 2a4b9f0: scaffold controller, policy enums, and feature-flag checks
- 7f3d1a2: integrate controller into scheduler dispatch path with soft-throttle
- 9d4c8b1: kv_cache staleness grace windows + metrics
- b2a9e0c: integration tests + replay harness
- c3f7f55: docs and canary manifests
- f4a6da1: benchmark scripts and baseline reports

Files changed (high level):
- runtime/clc_controller.cc
- runtime/clc_controller.h
- runtime/scheduler.cc
- runtime/scheduler.h
- runtime/kv_cache_manager.cc
- runtime/kv_cache_manager.h
- tests/clc_integration_test.py
- tests/unit/test_clc_policy.py
- docs/operational/clc.md
- deployments/canary/clc-canary.yaml
- benchmarks/clc_canary_report.md

Review notes and conversation: "Lena Zhao: overall this looks good; controller state machine LGTM but please add a metric for 'policy_flip_rate' so we can alert on noisy decisions." "Asha Patel (author): added policy_flip_rate metric exposed via telemetry.v2 in 7f3d1a2; also added smoothing window to avoid flip storms." "Samir Gupta: scheduler changes are sensitive; I ran long-running internal perf tests on a small fleet and saw no regression. Suggest we add a toggle to force slow-path for a single model id for debugging." "Asha Patel: added per-model debug toggle in clc_controller.h and documented usage in runbook." "Miguel Torres (approver): verified canary numbers, approved for squash merge."

CI summary: "unit-tests: pass; integration-tests: flaky -> rerun -> pass; lint: pass; benchmarks: pass; canary-smoke: pass."

Merge outcome: "Merged via squash on main by Miguel Torres on 2025-02-08. Canary rollout configuration added to deployment manifests. No breaking API changes; default behavior is opt-in behind feature flag."

Known limitations and follow-ups: "1) The current output-diff signal is syntactic (token-level divergence) and can produce false positives for harmless rearrangements; follow-up ENG-8602 will add semantic eval sniffers. 2) We do not yet persist controller state across restarts; follow-up ENG-8603 will add lightweight state checkpointing. 3) Need dashboards for policy_flip_rate and fastpath_denied_ratio (ENG-8610)."
Lena Zhao: overall this looks good; controller state machine LGTM but please add a metric for 'policy_flip_rate' so we can alert on noisy decisions.
Asha Patel: added policy_flip_rate metric exposed via telemetry.v2 in 7f3d1a2; also added smoothing window to avoid flip storms.
Samir Gupta: scheduler changes are sensitive; I ran long-running internal perf tests on a small fleet and saw no regression. Suggest we add a toggle to force slow-path for a single model id for debugging.
Asha Patel: added per-model debug toggle in clc_controller.h and documented usage in runbook.
Miguel Torres: verified canary numbers, approved for squash merge.
Introduce Correctness-Latency Controller (CLC): opt-in runtime component to balance correctness and latency by adaptive precision toggles, fast-path soft-throttling, batching window overrides, and KV staleness grace. Feature flag gated; behind canary rollout.
