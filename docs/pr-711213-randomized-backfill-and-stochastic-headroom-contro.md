# Introduce randomized backfill and stochastic headroom controller

Background and motivation: We observed repeated headroom thrashing during moderate sustained bursts where deterministic backfill caused oscillation between warm and cold pools (see SUP-4123 incident notes). This PR implements two coordinated features: 1) an epochal randomized backfiller that staggers warm-pool replenishment across epochs to reduce correlated allocation spikes; and 2) a stochastic headroom controller that computes per-tenant probabilistic headroom allocations based on recent utilization jitter and a configurable confidence parameter. Together these reduce momentary oversubscription and improve fairness for small tenants while preserving throughput guarantees for committed capacity.

What changed (high level):
- autoscaler/epoch_backfill.go: new epochal backfill controller with randomized jitter and backoff windows.
- autoscaler/headroom_controller.go: new stochastic allocation engine that emits allocation proposals with an attached confidence score and TTL.
- autoscaler/hooks.go: integration points for the warm-pool reconciler to accept epochal backfill proposals.
- telemetry/utilization_fairness.md: documentation for the new fairness metrics and labels added to metrics emission.
- metrics/exporter.go: new metrics endpoints: redwood_autoscaler_headroom_proposal{tenant,confidence}, redwood_backfill_epoch_active, and a percentile sampler redwood_utilization_jitter_p50/p90.
- config/defaults.yaml: added knobs: epoch_backfill.window_ms, epoch_backfill.jitter_seed, headroom.confidence_default, headroom.max_ttl_ms.
- e2e/autoscale_backfill_test.go: deterministic test harness with synthetic tenants exercising staggered replenishment.
- docs/design/epochal-backfill.md: design rationale, failure modes, and rollout plan.

Implementation notes: The backfiller runs on an epoch boundary (default 10s) and computes a target set of warmlets to prewarm. Instead of issuing all prewarm requests immediately, it samples a shuffle offset derived from a tenant-scoped PRNG seeded by epoch_id xor jitter_seed. This ensures consistent but desynchronized replenishment across minutes without global coordination. The headroom controller ingests sliding-window utilization deltas and fits a lightweight asymmetric quantile estimator to produce a conservative proposal at the requested confidence. Proposals are advisory; the admission controller still validates hard reservation constraints.

Testing and CI: Unit tests cover PRNG behavior, headroom proposal math, and edge cases (empty history, sudden drop to zero). e2e tests simulate 100 tenants with mixed commit levels and assert that 95th-percentile cold-starts are reduced by ~25% in the synthetic workload. Initial CI run showed a flaky e2e related to timing; fixed by replacing time.Sleep-based synchronization with epoch barrier signals. All CI checks are green on the final push.

Rollout and safety: Feature-gated by config.features.epochal_backfill and headroom.enabled. Default is off for existing clusters. Rollout plan: canary on low-traffic pool for 24h, then incremental ramp to 25% of warm pools, monitor redwood_autoscaler_headroom_rejects and redwood_cold_start_95. If regressions are observed revert via runtime flag or return to previous commit.

Checklist:
- [x] Unit tests for PRNG and headroom math
- [x] e2e reproducible harness
- [x] Metrics and dashboards updated
- [x] Docs and rollout plan included
- [x] Feature gate behind config flag

Related work and notes: This is complementary to the warm-pool smoothing work (see ENG-1983) but focuses on de-correlation rather than smoothing heuristics. Linear items: ENG-2147 (design), ENG-2204 (telemetry). Support ticket SUP-4123 provided incident traces that motivated desynchronization.

Commits summary: "epochal backfill controller initial implementation"; "stochastic headroom controller and proposer interface"; "metrics: headroom proposal and jitter percentiles"; "tests: deterministic epoch harness + e2e scenarios"; "docs: design + telemetry".

Changed areas/files (high level):
- pkg/autoscaler/epoch_backfill.go
- pkg/autoscaler/headroom_controller.go
- pkg/autoscaler/hooks.go
- pkg/telemetry/utilization_fairness.md
- pkg/metrics/exporter.go
- config/defaults.yaml
- test/e2e/autoscale_backfill_test.go
- docs/design/epochal-backfill.md

Review conversation (summary):
- Raj Patel (comment): "The PRNG seeding looks good but we should ensure reproducible results for replay tests — can we expose the jitter_seed as a test-only override?"
- Lena Park (reply): "Added jitter_seed override in config and test harness; updated tests to set a deterministic seed."
- Marta Alvarez (request_changes): "The headroom proposal includes a confidence float; we need a defensive clamp to avoid NaN when history is zero-length. Also add a metric for proposals rejected by admission controller so we can monitor mismatch between advisory and enforced constraints."
- Lena Park (reply): "Added clamp logic and proposed_rejects metric. Extended unit tests to cover zero-length windows and admission rejects."
- Owen Chen (approve): "Code looks clean, and tests exercise both happy-path and edge cases. I like the epoch barrier approach for e2e stability."
- Raj Patel (approve): "Verified canary run on staging for 6h; no regressions observed. Metrics show improved 95th cold-start latency under synthetic churn."

CI details: Build: passed; Unit tests: passed; Linters: passed; e2e: passed (after fix); Re-run of flaky job was manual with timestamped barrier.

Merge: merged via squash on 2026-03-01 by Lena Park. Merge commit message prefixed with "feat(autoscaler): epochal randomized backfill + stochastic headroom".

Notes for on-call: Watch metrics redwood_cold_start_95, redwood_autoscaler_headroom_proposal_rejects, redwood_backfill_epoch_active. If warm-pool oversubscription is observed, flip config.features.epochal_backfill=false and allow the autoscaler to reconcile.
