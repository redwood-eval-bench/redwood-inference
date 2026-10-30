# soft-warmpool-oracle-and-throughput-fallback-adapter

Motivation: Dedicated fleets still see short-lived microbursts that are expensive to satisfy with hard reservation policies. This PR introduces a lightweight WarmPool Oracle and a soft-capacity hedging adapter that allow fleets to accept limited temporary overcommit while preserving throughput guarantees. The goal is to reduce cold-start allocations and avoid aggressive scale-ups for short spikes, and to expose defensible telemetry so SRE and customers can tune hedging windows.

What this PR does: 1) Adds a WarmPool Oracle service that maintains a compact probabilistic model of warmlet readiness per pool (warm_index, recent-prewarm-rate, drain-backoff). 2) Introduces a SoftCapacityHedge admission adapter in the fleet allocator that can issue provisional credits (soft_burst) with configurable TTL and prioritized fallback routing when hedges are exhausted. 3) Implements throughput-fallback contracts: when a soft hedge cannot be honored, requests are rerouted to a lower-cost compatible model variant or throttled according to priority bands with deterministic backoff. 4) Emits new telemetry and SLIs- a per-pool warm_index gauge and a per-tenant hedged_admissions counter plus percentile-backed latency impact histograms. 5) Adds an opt-in feature flag (enable_warmpool_oracle) and rollout plan (canary on us-west-2 followed by gradual region expansion).

Checklist: - [x] Design doc linked in PR (ENG-2145 design) - [x] Unit tests for oracle state transitions - [x] Integration tests for hedge expiry and fallback routing - [x] E2E load test with synthetic microburst traces - [x] Docs update (console UI & config reference)

Backwards compatibility and safety: All new behavior is behind the feature flag; soft hedges are bounded by a configurable safety cap (soft_hedge_cap_per_pool) to avoid systemic overcommit. Fallback routing uses the existing model-variant affinity rules so we do not introduce new model incompatibilities.

Rollout notes: Start with canary enabled for dedicated pools in us-west-2 at 10% of traffic for one week. Monitor redwood_throughput_guarantee_violations_total and warmpool_oracle_warm_index percentiles; abort if guarantees spike > 0.5% over baseline.

Related: ENG-2145, ENG-2190, SUP-1122.
Maya Chen: Nice design. A few questions: how do you ensure the oracle model cannot be gamed by noisy tenants? Suggest adding an upper bound on soft_hedge credits per tenant.
Author (Ishaan): Added soft_hedge_cap_per_tenant and documented in config. Oracle uses exponentially damped counters to avoid short-term amplification.
Diego Rivera: Please add a unit test for oracle state 'stale->refill' transition under concurrent updates.
Author: Added TestOracleRefillConcurrent in warmpool_test.go and a race-check CI job for the integration stage.
Priya Nambiar: Telemetry names look good. Can we tag hedges with pool_id and particle (prewarm) bucket for easier dashboards?
Author: Telemetry now includes pool_id and bucket label; dashboards updated in console commit.
