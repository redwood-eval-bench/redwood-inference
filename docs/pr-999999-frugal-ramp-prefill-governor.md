# Introduce frugal ramp prefill governor and lightweight econ presets

Motivation: many tenants exhibit high spend during short bursty onramps because the existing prefill strategy starts with a generous KV-prefill and conservative quantization, which increases token cost for the first N requests. This PR introduces a "frugal ramp prefill governor": a lightweight runtime policy that gradually scales prefill aggressiveness and quantization-presets over a short ramp window. Goals: reduce first-minute unit cost by ~30% for low-latency, bursty workloads without compromising observed user latency SLOs; provide tenant-level safe defaults that require no operator tuning; make the governor pluggable into Dedicated and Private deployments.

Changeset overview: a) runtime: new governor implementation in serving/runtime/prefill/frugal_ramp_governor.go, with a simple state-machine that tracks ramp age and recent token-per-request distribution; b) config: a new tenant-level econ profile preset (configs/econ/presets/frugal-ramp.toml) with three progressive quantization suggestions and prefill multipliers; c) orchestration: small API surface to expose the governor parameters via the Console and CLI (api/tenant_prefill.go, console/ui-prefill.tsx); d) safeties: added regression guard in observability to compare quality metrics for the first 100 requests post-ramp and auto-roll back to conservative profile on degradation (observability/alerts/prefill_regression.yaml); e) docs: updated quickstart and runtime notes (docs/ops/prefill-governor.md).

Checklist:
- [x] Unit tests for governor state transitions and thresholds
- [x] Integration test for tenant-level runtime wiring on dedicated cluster
- [x] Telemetry hooks added for econ metrics (tokens_saved, prefill_hits, rollback_count)
- [x] Console controls: toggle + non-admin defaults documented
- [x] Safety regression guard + automatic fallback
- [x] Release notes drafted

Commits (high level):
- 1a2f3b4: initial governor impl + interfaces
- 2b3c4d5: config presets and TOML schema updates
- 3c4d5e6: console UI and API endpoints
- 4d5e6f7: telemetry + alert rules
- 5e6f7g8: tests and QA tweaks

Changed files (high level):
- serving/runtime/prefill/frugal_ramp_governor.go
- configs/econ/presets/frugal-ramp.toml
- api/tenant_prefill.go
- console/ui-prefill.tsx
- observability/alerts/prefill_regression.yaml
- docs/ops/prefill-governor.md

Notes on design decisions:
- We intentionally keep the governor state machine minimal (age buckets: cold, ramp, steady) to avoid adding global scheduling complexity. The governor exposes a per-tenant budget token that decays with time spent in cold state.
- Progression of quantization presets is conservative: FP16 -> int8-dynamic -> int8-static (when model and hardware advertise compatibility). The system only advances quant modes when quality checks for generated tokens pass a light classifier (evals/quality_light.py) aggregated over samples.
- Fallback is immediate if the telemetry alert detects sample-level perplexity/RoBERTa BLEU regressions beyond configured thresholds.

Performance and safety validation:
- Staging synthetic load test (tenant-snapshot-88) shows median token cost drop of 28% in the first 60s and 3–5% overall throughput increase due to lighter initial KV prefill paths.
- Observability simulated regression tests exercised automatic rollback and verified no lingering degraded traffic after rollback.

Rollout plan:
- Canary rollout to 5% of tenants for 48h (schedules: dedicated and hosted regions us-west1/eu-west1), monitor tokens_saved, rollback_count, latency P99; if no regressions, advance to 50% over 72h.

Release notes (for changelog):
"Introduce frugal ramp prefill governor: tenant-safe default that reduces initial-token cost for bursty workloads while preserving latency SLOs. Includes console options and automatic regression fallback."

Merge outcome: Squash-merged into main after approvals and green CI
Jordan Kim: Please add a unit test covering the 'fast rollback' scenario where quality fails after ramp transition.
Author Asha Patel: Added test and adjusted rollback cooldown to 30s to avoid flapping.
Liam O'Connor: Concerned about advancing quantization too aggressively on older GPU families — added guard to only allow static int8 on declared-capable hardware.
Priya Rao: UI copy looks good; suggest adding an explanatory tooltip about tokens_saved calculation.
Jordan Kim: CI shows a flaky regression in integration/test_prefill_on_dedicated — rerunning.
Frugal ramp prefill governor: tenant-safe defaults to reduce first-minute token costs for bursty traffic; console toggle and automatic rollback on quality regressions.
QA ran the dedicated cluster tenancy test, observed rollback firing and reverting to conservative prefill. No user-visible latency regression seen in canary window.
kick off canary: tag canary/5pct and instrument tokens_saved dashboard
notify-ops: run weekly review after canary 48h
docs: community quickstart updated
No immediate backport to release/* required. If a customer ondedicated wants earlier access, branch off release-2025.10 and apply frugal_ramp_governor.go and preset TOML only.
