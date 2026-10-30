# Soft SLA contingency router and usage reconciliation pipeline

Motivation: tenants on Dedicated occasionally approach committed latency SLOs during short spikes. Current strict admission controls either throttle aggressively (hurting tail latency) or immediately fall back to hosted variants (increasing cost and losing isolation). This PR introduces a "soft SLA contingency router" that probabilistically diverts a tunable fraction of incoming requests to reserve lanes or to hosted fallback based on short-term predictive headroom and tenant contract thresholds, combined with a nightly "usage reconciliation" pipeline that compares billed reservation credits to observed GPU utilization to surface over/under allocation and generate recommended reservation adjustments.

This is a production-focused change: behavior is off by default and gated behind a feature flag (ops.contingency.enabled). Main goals are: 1) preserve critical tenant latency SLOs gracefully during microbursts, 2) avoid wholesale fallback unless necessary, 3) provide visibility and automated recommendations to correct reservation drift.

What I changed (high level):
- runtime/controller/contingency_router.go: new router component implementing soft diversion policy, short term headroom estimator, and per-tenant contingency token accounting.
- pkg/sim/sla_simulator.go: lightweight deterministic simulator used by unit tests and can be run locally to validate routing behavior under synthetic burst patterns.
- ops/reports/usage_recon.py: nightly reconciliation job that aggregates per-GPU utilization samples, cross-references reservation ledgers, and emits recommended reservation deltas to Linear.
- docs/ops/contingency_routing.md: operator runbook (feature flag, metrics, rollback).
- test/unit/contingency_router_test.go: deterministic tests for token accounting, diversion probability bounds, and concurrent update safety.
- charts/redwood/values.yaml: small values to expose feature flag to Helm for Private installs.

Checklist: [x] design doc linked (ENG-1412) [x] unit tests added [x] e2e smoke script (manual) [x] feature flag gated [x] docs & runbook updated [ ] migration script (not required; migration is zero-downtime)

Notes on behavior and safety: The router maintains a sliding-window headroom estimate per fleet (default 30s window) and computes a diversion ratio r(t) = clamp(0,1, alpha * (1 - remaining_commit_fraction)). Remaining_commit_fraction uses an exponentially decayed observed utilization vs reservation baseline. When r(t)>0 the router probabilistically routes requests: some to dedicated reserve lanes (if tokens available), some to hosted fallback (if configured), otherwise to a best-effort queue. Contingency tokens are sourced from a soft reserve ledger that decays when used and replenishes at a configurable rate to avoid persistent stealing from other tenants.

Monitoring and alerts: New metrics are emitted: contingency.router.diversion_ratio, contingency.router.tokens_consumed, usage_recon.recommendation_delta_count, usage_recon.unmatched_samples. Dashboards and alerts are documented in docs/ops/contingency_routing.md.

Migration & rollout plan: feature flag off by default. Canary rollout to two small fleets for 48h, validate contingency.router.tokens_consumed < 5% of committed capacity and nightly usage_recon false-positive rate < 2%. If metrics spike, operator can toggle ops.contingency.enabled=false and abort.

Related: ENG-1419 covers reconciliation alerting thresholds. SUP-827 flagged discrepancy cases that motivated the reconciliation job.

Aisha Kumar (reviewer): Nice direction. Two immediate concerns: 1) race on token ledger increments when many goroutines try to claim; please add CAS/compare-and-swap or a per-shard mutex. 2) metrics are great but we need an SLA-preserving alert if diversion_ratio>0.3 for >5m.
Maya Chen (author): Added per-shard mutex in commit b4c5d6e and exposed contingency.router.diversion_ratio alert with 5m burn-in. Also tightened diversion cap to 0.4 from 0.6 per product request.
Daniel Park (reviewer): Unit tests look good. Could we add one integration smoke that runs simulator + reconciliation job? Concerned about nightly job cost scanning all samples.
Maya Chen: Reworked reconciliation to sample at 1m granularity and sample 10% of GPUs by default. Added integration smoke script under scripts/smoke/contingency_smoke.sh (not included in this PR) and documented cost tradeoffs.
Luca Romano (approver): Verified metrics on canary fleet. Small nits: document helm flag location and add release note. Approving pending doc tweak.
Maya Chen: Addressed doc tweak and added release notes. Ready to merge.
Add a feature-flagged soft SLA contingency router that probabilistically diverts requests to reserve lanes or hosted fallback during microbursts; add nightly usage reconciliation job that recommends reservation adjustments based on observed GPU utilization. Operator docs and metrics included.
