# Introduce ephemeral SLO gradient normalizer and rate-cap smoothing to reduce circuit flapping

Context and motivation: In recent incidents we observed rapid SLO re-evaluations at the regional level that produced near-simultaneous circuit openings and aggressive shedding. This PR adds a small, conservative runtime layer that smooths transient SLO gradients and a rate-cap smoothing middleware at the ingress path to reduce instantaneous pressure spikes. The design goal is to lower the probability of cascading circuit trips while preserving prompt reaction to sustained degradation.

What this changes (summary): 1) runtime/slo: introduced `GradientNormalizer` which computes a short-timescale moving-window gradient of SLO error and emits a normalized smoothing factor that downstream enforcers can consume; 2) gateway/rate_limit: added `smoothing_middleware` that converts hard instant rate-cap deltas into temporally-smoothed adjustments using the normalizer value; 3) controlplane: telemetry events for normalized SLO signals and an operator-configurable safe-rollout flag; 4) tests and docs: unit/integration tests and an operational playbook for tuning the smoothing window.

Checklist: - [x] Design doc reviewed (ENG-6124) - [x] Unit and integration tests added - [x] Telemetry hooks instrumented - [x] Config flag and feature gate added for gradual rollout - [x] Canary plan documented in docs/operational/degradation_tuning.md

Testing notes: Initial CI surfaced a race in the normalizer when multiple region probes were run in quick succession; fixed by serializing the statistics collector with a lock and adding a deterministic test harness. Integration tests run across local multi-region simulator (5 regions) with both steady-state and spike workloads; results show 38% reduction in simultaneous circuit openings in the simulator with 4x synthetic spikes. Load tests validated no measurable tail latency regression under normal traffic; p95/p99 unchanged within noise.

Rollout plan: 1) Feature-flag off by default; release to canary (10% of capacity) for 48h. 2) Collect normalized_slo and circuit_open_rate metrics, evaluate for degradation or unexpected latencies. 3) Enable rate-cap smoothing at ingress for canary, then ramp to 50% for three days, monitor. 4) Full rollout if stable. Rollback is dropping the feature flag and restarting the middleware deployment; this PR includes a safe config map path for immediate disable.

Telemetry and alerting: New metric `slo.normalized_gradient` (double) and `slo.smoothing_factor` (0..1). Alert rule suggestions included in deploy/alerts/slo_smoothing_rules.yaml.

Risks and mitigations: Smoothing too aggressively could delay reaction to real outages. We kept the normalizer conservative (short window, tunable), and we added a hard override so sustained SLO breaches still trigger circuit logic after a configurable threshold.

Related: ENG-6124 (design), ENG-6140 (canary plan).
Adds a short-lived SLO-gradient normalizer and a rate-cap smoothing middleware to mitigate abrupt circuit trips across regions. Includes telemetry hooks and a safe rollout gate; reversible via config flag.
Diego Morales: Please justify the chosen smoothing window size; could we make it adaptive to traffic variance?
Aisha Patel: Answered - kept fixed short window by default for stability, added config knob and comment explaining adaptive mode is future work (ENG-6201).
Juniper Lee: Found a race during stress test where collector map was mutated without lock. Recommend serializing updates or using sharded locks.
Aisha Patel: Fixed in commit d4f6a2c; added simple mutex around stats collector and deterministic unit test to prevent regression.
Evelyn Chen: Can we ensure the smoothing middleware never increases rate caps above what the allocator intends (safety invariant)?
Aisha Patel: Confirmed middleware only scales downward smoothing factors; added explicit invariant check and a unit test asserting allocator cap is an upper bound.
Juniper Lee: Approving pending CI green
Diego Morales: Approved
Evelyn Chen: Approved - left a follow-up to consider adaptive windowing after rollout (ENG-6201).
