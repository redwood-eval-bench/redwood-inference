# reservoir-based-tenant-quotas-with-leaky-recovery-and-eviction-safety

Introduce a reservoir-based per-tenant quota allocator with leaky-bucket style recovery and eviction-safety. Adds metrics and migration guidance. Backwards compatible with current token pools.
Context: Dedicated clusters sometimes see oscillatory tenant throughput after hard evictions or burst draining: tenants either starve for a long tail or spike immediately when capacity frees up. This PR implements a reservoir-based quota allocator that smooths tenant replenishment and prevents transient takeover while preserving burstability.

What this implements: a tenant 'reservoir' (soft budget) that is consumed by requests. The reservoir refills at a configurable leak rate (tokens/sec) computed from a tiered policy: base_rate (guaranteed throughput), adaptive_boost (short-term refill after sustained idleness) and cooldown_decay (prevents aggressive refills after eviction). Replenishment is computed as a convolution over a short moving window of measured served tokens, resulting in a gentle recovery curve rather than an instantaneous refill.

Key features and rationale: 1) Smooth recovery curve: avoids sudden token floods after preemption or cold-start; improves tail latency for co-tenants. 2) Eviction-safety: when a tenant is preempted, reservoir recovery follows a delayed ramp (configurable) to avoid immediate reconsumption of freed capacity. 3) Compatibility: the allocator sits as an optional policy in the Dedicated scheduler and falls back to existing percent-based quotas when disabled. 4) Observability: exposes reservoir_fill, reservoir_drain, refill_rate, and overdraw events with histograms for recovery time. 5) Admin controls: per-tenant profiles (steady_rate, burst_cap, recovery_delay, cooldown_mult) and a cluster-wide override for emergencies.

Design notes: reservoirs are represented as a floating point budget tracked in the scheduler; requests check budget before admission. Overdraws are allowed up to a small safety cap (configurable) and are penalized by lengthening the cooldown. The replenishment formula is intentionally conservative (90th percentile pacing) to prioritize SLO stability over aggressive utilization. We also added a short-term credits buffer to allow headroom for in-flight KV-cache misses without immediate throttle.

Testing and validation: unit tests for refill math and edge cases (eviction+immediate traffic) included. Integration (e2e) tests simulate a 3-tenant dedicated cluster with one tenant evicted mid-run and verify no more than 10% cross-tenant tail latency degradation. Perf runs on staging show a 12% reduction in 99p latency for co-tenants in eviction scenarios and negligible steady-state throughput loss (~0.8% median).

Migration plan: feature-flag rollout (off by default). Stage 1: enable in canary clusters for 48h, collect metrics and alarms. Stage 2: opt-in for customers on Dedicated beta. Stage 3: default enable after two weeks of stable signals. We included a toggle to map existing token-pool quotas to initial reservoir levels to avoid surprise changes.

Checklist: - [x] Core allocator implementation - [x] Config schema and defaults - [x] Metrics + dashboards - [x] Unit + integration tests - [x] Docs/ops runbook - [x] Feature flag gating

Risks and mitigations: incorrect refill tuning could reduce utilization — we default to conservative rates and provide easy overrides. Eviction edge cases were stress-tested; added telemetry to detect recovery stalls.

Files of interest (summary): see files_changed for details.

Notes for reviewers: focus on refill math in pkg/autoscaler/reservoir.go and on the eviction handoff path in internal/scheduler/alloc.go. Metrics naming follows the new observability conventions and is backwards compatible with dashboards in observability/metrics.

Liam Becker (2025-02-13): Can we get a clearer justification for the default cooldown_mult=1.5? Suggest adding a short doc explaining the numeric choice and sensitivity tests.
Aisha Patel (2025-02-13): Added sensitivity tests and a section in docs/ops/reservoir-quotas.md explaining experiments. Default chosen to limit immediate reconsumption while allowing quick steady-state recovery.
Marco Chen (2025-02-14): Code looks good; please add a small e2e variant where multiple tenants evict sequentially to ensure we don't cascade penalties.
Aisha Patel (2025-02-15): Added e2e scenario 'sequential-eviction' and adjusted cooldown backoff logic to be multiplicative across closely-spaced evictions.
Priya Rao (2025-02-16): Approved after verifying metrics and test outputs in staging (099-staging). Left minor nits on logging verbosity.
Liam Becker (2025-02-17): Approve — verified perf numbers and that feature flag toggles cleanly without state corruption.
