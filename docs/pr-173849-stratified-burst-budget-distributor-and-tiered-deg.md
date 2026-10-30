# Stratified burst budget distributor and tiered degradation plan for Dedicated fleet

Motivation: Tenants on Dedicated fleets need more predictable degradation and fairer use of limited burst credits during transient demand spikes. Current global soft-burst pools provide throughput but cause noisy neighbor effects and opaque degradation patterns. This change introduces a stratified burst budget distributor and a tiered degradation plan to give operators fine-grained control over how burst capacity is allocated and how tenants degrade when capacity is constrained.

What this PR does: Implements a new budget distributor that divides burst credits into named strata (critical, standard, opportunistic) and allocates credits to tenants based on configurable weights, recent utilization percentiles, and tenant commitment tiers. Adds an anticipatory refill algorithm (EWMA-based arrival estimate + percentile headroom multiplier) so short-lived surges are smoothed without inflating steady-state consumption.

Key features: 1) Config-driven strata: fleet operators can define strata names, target percentile thresholds, and per-strata reclaim priorities. 2) Tenant scoring: runtime computes a transient score per tenant combining weight, p50/p95 utilisation deltas, and recent burst consumption to determine which strata they draw from. 3) Tiered degradation plan: when fleet-wide capacity approaches configured limits, the plan enacts gradual degradation by stepping tenants down from opportunistic -> standard -> critical; critical tenants retain minimum headroom guarantees. 4) Safety and rollout: change is gated behind feature flag tenant_strata_rollout (off by default). Existing behavior is preserved unless strata are configured. 5) Telemetry: new metrics and aggregated reports to observe per-strata balances and tenant degradation events.

Implementation notes: - New core components: pkg/autoscaler/stratified_distributor.go, pkg/autoscaler/strata_policy.go - Extensions to DedicatedCapacitySpec (api/v1beta1/dedicated.proto) with clear validation rules and defaults. - Telemetry instrumentation added in telemetry/metrics.go and observability/alerts.yaml to surface regression in p95 latency when degradation steps occur. - Unit + integration tests cover allocation invariants, fallback to legacy global pool, and degradation step ordering. - A perf-canary job (benchmark-lab/canaries/strata-10k-tenants.yaml) demonstrates behavior under mixed commitment cohorts.

Backward compatibility and migration: The DedicatedCapacitySpec changes are additive. If burstStrata is not set, the distributor falls back to the legacy single-pool allocator. All proto changes are optional fields and include server-side defaults. The rollout plan documented in docs/dedicated/strata-rollout.md prescribes a staged enablement: internal canary -> opt-in customers -> graduated default.

Operational considerations: - New metrics: redwood_strata_credit_balance{strata}, redwood_tenant_burst_balance{tenant}, redwood_strata_eviction_events{strata} - Alerting: add SLO guard for rapid p95 regressions after enabling strata on a fleet (see observability/alerts.yaml) - Suggested knob values and a sample policy are included in examples/strata-sample-policy.yaml

Related work: builds on prior autoscaler telemetry plumbing (ENG-2310) and observer-based reservation contracts (ENG-1823).

Testing: comprehensive unit tests added (pkg/autoscaler/*_test.go), integration tests in test/integration/strata_integration_test.go, and a load canary in benchmark-lab. CI initially hit a flaky integration test exercising time.Sleep-based refill; I replaced the time dependency with a controllable clock in the distributor and re-ran CI.

Merge notes: squash-merge after sign-off from runtime and SRE.

Priya: Can the distributor revert to legacy behavior per-fleet?
Maya: Yes, default is legacy single-pool; strata only active when configured. Added a unit test to assert fallback.
Ethan: The refill EWMA constant looks aggressive; can we expose it as a knob?
Maya: Exposed as fleet-level parameter with sensible default and documented recommended ranges.
Lars: Add more observability around tenant step-down events so SRE can correlate regressions.
Maya: Added redwood_strata_eviction_events and a dashboard link in docs.
Priya: CI flake unrelated to logic (timing). After swapping to controllable clock we get green. Approving.

Introduce stratified burst budgeting and configurable tiered degradation for Dedicated fleets. Adds new DedicatedCapacitySpec fields (burstStrata, degradationPolicy) and telemetry metrics. Backwards compatible behind a rollout feature flag.
