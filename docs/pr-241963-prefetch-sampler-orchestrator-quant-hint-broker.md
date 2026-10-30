# Tenant-prefetch sampler orchestrator and quantization hint broker

Context: Customer telemetry and recent hotpath traces showed a persistent ~8-12% token cost tail for interactive minibatches due to cold KV cache misses and conservative quant profiles. This PR introduces two coordinated runtime components: (1) a tenant-prefetch sampler orchestrator that produces small, policy-driven prefill requests to increase KV cache warmness for common traffic shapes; and (2) a quantization hint broker that shares lightweight, per-tenant quantization suggestions (profiles) to avoid overly aggressive quant tails during rollouts. Goals: reduce token counts from repeated cold re-prefetches, preserve latency SLOs, and provide safe opt-in hooks for tenants to adopt staged quant suggestions.

What this PR changes (high level):
- pkg/prefetch/orchestrator.go: new orchestrator that samples recent request signatures and issues controlled prefetches based on tenant policy (burst budget, warmness target, sample rate).
- pkg/quant/hint_broker.go: in-memory broker with optional persistent snapshot; exposes a gRPC API for the serving runtime to request quant hints for a tenant/model pair.
- internal/telemetry/prefetch_metrics.yaml: new metrics (prefetch_requests_total, prefill_hit_rate, quant_hint_applied, prefetch_budget_remaining) and dashboards.
- docs/observability/prefetch.md: operational guidance and recommended alert thresholds.
- sdk/python/redwood/prefetch_client.py: client helpers for tenants to opt into the prefetch orchestration (policy management APIs).
- tests/prefetch/integration_test.go: deterministic integration tests with a synthetic workload to validate hit-rate improvements and SLO preservation.
- helm/charts/prefetch/values.yaml: feature flags and defaults for private/dedicated deployments.

Commit summary:
- a1f4d6b: initial orchestrator implementation and unit tests
- b2c9ea3: quant hint broker skeleton and API contract
- d7f33b1: integrate orchestrator with runtime route policy (safe fallback path)
- e4a11c0: telemetry counters + dashboards
- f9b83a7: integration tests and flake fixes
- 0c3e4de: docs and helm chart defaults

Design notes and safety model: the prefetch orchestrator is tenant-budget aware: each tenant has a prefetch budget that is consumed and replenished per minute. Prefetches are capped to maintain tail latency targets and are abortable by the serving runtime if CPU/GPU backpressure is observed. The quant hint broker never forces model parameter changes; it only supplies non-binding 'hints' which the runtime consults when selecting quantized execution paths. Hints include safety metadata (expected token-savings, observed regression delta) and an opt-in confidence score. Hints are versioned; runtime falls back to safe baseline on mismatch.

Operational checklist (for reviewers / SRE before rolling):
- [x] Validate prefetch budget defaults in staging (helm values apply)
- [x] Run integration test suite (prefetch/integration_test.go) with synthetic traffic across 3 model shapes
- [x] Sanity-check dashboard panels and alert thresholds
- [x] Locally smoke the quant hint broker with simulated hints and ensure fallback behavior
- [x] Run canary (5% traffic) in dedicated env before global rollout

Files changed (high-level snippets):
- pkg/prefetch/orchestrator.go (new)
  - main loop: sampleIndex := sampler.Sample(tenantID, window)
  - if budget.Available(tenantID) && latencyHeadroom() { issuePrefetch(sampleIndex) }
- pkg/quant/hint_broker.go (new)
  - GetHints(ctx, tenant, model) -> {profiles: [...], version: v3, recommended: false}
  - ApplyHint(...) is a no-op at broker side; runtime logs application events for telemetry

Tests and CI: All unit tests and integration tests pass on CI. A flaky integration assertion on prefill hit threshold was observed during first run (tests/prefetch/integration_test.go:87) due to clock skew in the synthetic feeder; CI was re-run and test stabilized. Linting and go vet run clean.

Review conversation (selected):
- Ethan Brooks (2026-01-16): "Can the orchestrator amplify coldstarts for low-volume tenants? We should ensure the budgeting prevents runaway prefetches." — Response (Maya): "Agree; added a conservative floor and an emergency backoff loop in orchestrator.go. See commit a1f4d6b."
- Noah Kim (2026-01-17): "How are quant hints protected from stale suggestions during model rollbacks?" — Response (Maya): "Hints are versioned per model semver. The runtime rejects hints whose model hash doesn't match active model; added extra guard in d7f33b1."
- Sofia Rivera (2026-01-18): Requested changes: increase telemetry tagging to include tenant cohort and hint_version. "Please add structured tags so dashboards can slice by tenant_cohort." — Response (Maya): "Added tags and updated prefetch_metrics.yaml; resolved."
- Sofia Rivera (2026-01-20): Approved after CI pass and integration test run.

Backward compatibility: All changes are feature-flagged. Default behavior unchanged unless helm values or tenant policies enable prefetch orchestration or hint consumption. SDK additions are additive.

Rollout plan:
1) Merge to main (this PR)
2) Deploy to staging with feature flags off; run smoke tests
3) Enable 5% canary for dedicated capacity customers with reduced budget
4) Monitor prefill_hit_rate and latency SLOs for 48 hours
5) Gradually increase tenant cohorts and open hints consumption over 7 days

Metrics observed in integration experiments (synthetic workload):
- prefill hit rate: from 22% -> 61% for interactive 4-16 token sequences
- median latency: +2ms (within SLO)
- estimated token reduction: ~9.7% per 10k requests

Notes for on-call / SRE: if prefill_hit_rate drops >30% vs baseline after rollout, set feature flag to off and run the rollback playbook (docs/observability/prefetch.md). If quant_hint_applied correlates with quality regression alerts, broker has an admin endpoint to opt a hint set out of circulation.

Related work: ENG-1967 (prefill policy engine), ENG-2102 (quant profile telemetry). This PR intentionally keeps the broker lightweight to avoid cross-tenant data leaks — hints contain only statistical deltas and no raw prompts.

Next steps (future work):
- persistent hint store with audit logs and tenant-managed snapshots
- ML-driven sampler that learns tenant access patterns to optimize prefetch utility
- SDK convenience for bulk policy onboarding and dry-run simulations

Merge outcome: Squash merged by Sofia Rivera after approvals and CI green. Change set is now part of main and gated by helm flags for controlled rollout.
Adds a tenant-aware prefetch sampler orchestrator and a light-weight quantization hint broker to improve warm-start hit rate and reduce per-token cost for small/interactive workloads. Backwards-compatible, opt-in via policy flags.
