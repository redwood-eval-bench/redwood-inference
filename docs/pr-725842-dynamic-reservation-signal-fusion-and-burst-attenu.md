# Dynamic Reservation Signal Fusion and Burst Attenuator

Motivation: Dedicated fleets occasionally exhibit short, high-amplitude bursts that consume ephemeral reservation credits and cause downstream latency spikes for latency-sensitive tenants. Existing autoscaler signals are slow to react (windowed utilization) and telemetry reporters surface utilization in isolation, making coordinated soft-degradation difficult. This PR implements a lightweight signal-fusion layer and a burst attenuator that cooperates with the reservation allocator to smooth transient demand and protect throughput guarantees.

What this changes: Introduces a 'reservation-signal-fusion' subsystem that consumes telemetry (token occupancy, per-tenant in-flight counts, warm-pool pressure, recent allocation churn) and emits a short-lived derated target reservation and an attenuation score per tenant. The burst attenuator uses that score to temporarily reduce per-tenant soft-burst multipliers and coordinate with the warm-pool preemption policy so we avoid tail latency regressions during microbursts.

Key behaviors (configurable knobs):
- fusion_window_ms (default 1500): timespan over which signals are fused to produce an attenuation score.
- attenuation_decay_ms (default 30000): how quickly attenuation decays back to zero.
- min_soft_multiplier (default 0.6): floor for soft-burst reductions when attenuation is high.
- publish_mode: 'summary' | 'sampled' (summary by default) controls telemetry emission.

Backward compatibility: All knobs are feature-flagged under feature.dynamic_reservation_fusion. When disabled the system is a no-op and existing reservation logic is unchanged. No API changes to external endpoints. Operator-visible config is added to dedicated cluster config (yaml additions noted below).

Performance/impact: In a 48h synthetic microburst test (70 tenants, staged fanout), enabling fusion + attenuator reduced 95th token latency for affected tenants by 12% and lowered transient warm-pool churn by 18% while increasing steady-state provisioning latency by <1%.

Rollout plan: start as opt-in in dedicated canary pool for 2 weeks, enable per-cluster telemetry sampling, and then a staged rollout across fleets with 10% increments and health checks for per-tenant SLO regressions.

Testing: unit tests for signal fusion logic, end-to-end integration test that simulates rapid burst patterns, and a load test scenario added to perf-canary harness. Telemetry assertions added to observability-pack dashboards.

Related: ENG-4523 (design), ENG-4601 (telemetry adapter), SUP-128 (support ticket reporting burst regressions).
Aidan Morales (review): Can we avoid introducing another periodic loop for fusion? If possible prefer event-driven updates to keep CPU low on large clusters.
Author (Marisol): Implemented a hybrid approach: events schedule a fusion run, but we also fall back to a lightweight timer to drain late-arriving signals. See engine.go comment block for rationale.
Priya Nandakumar (review): The attenuation_decay_ms seems long for some workloads; suggest making decay a percentile function of recent request interarrival times.
Author (Marisol): Converted decay to 'decay_policy' that supports 'fixed' (default) and 'adaptive-percentile' (opt-in). Added tests and documentation for policy selection.
Lucas Rivera (review): Observability - I want a per-tenant attenuation metric exposed to the console and an alert if attenuation > 0.9 for > 2m.
Author (Marisol): Added attenuation_metric (gauge) and an example alert rule in docs. Also added a dashboard tile in observability-pack.
CI (automated): Initial run failed -- e2e perf scenario timed out intermittently (flaky).
Author (Marisol): Re-ran e2e with an increased timeout and added retry for a flakey test harness allocation step. All checks green.
Aidan Morales: Approve after addressing event-driven concern and adding docs.
Priya Nandakumar: Approve.
Lucas Rivera: Approve (small nit in docs fixed).
Adds a feature-flagged 'reservation signal fusion' and burst attenuator to reduce microburst impact on dedicated fleets. Operator-configurable knobs and dashboards included. Rollout should be staged behind feature flag.
Enable feature.dynamic_reservation_fusion=false by default. For canary enable on a dedicated pool with low critical tenant load. Monitor metrics: fusion.attenuation_score, reservation.churn_rate, warmpool.preemption_rate, tenant.95p_token_latency.
Flip feature.dynamic_reservation_fusion to false and redeploy control-plane; attenuation is designed to be non-destructive and state decays within attenuation_decay_ms. If telemetry shows regressions disable immediately and open SUP ticket.
