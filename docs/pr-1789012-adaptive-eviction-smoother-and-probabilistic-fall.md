# Adaptive eviction smoother and probabilistic fallback heuristic to reduce oscillation during regional overloads

Motivation: In the last two oncall incidents (SUP-1182) we saw eviction+fallback-induced oscillations when a regional overload triggered aggressive tenant evictions and immediate rejoin attempts. That interaction caused repeated circuit flips and elevated 95th P50/P95 latency for affected tenants.

This PR introduces two complementary changes to reduce oscillation and provide smoother degradation behavior: 1) an Adaptive Eviction Smoother in the runtime scheduler which low-pass filters short-lived eviction pressure spikes and spaces eviction actions across a configurable window; 2) a Probabilistic Fallback Heuristic in the control plane which gates immediate regional failover by probabilistically delaying fallbacks for low-priority routes and gradually ramping fallback probability based on sustained error/latency signals.

What changed: single continuous description of design + checklist below. Key goals: preserve SLO enforcement semantics for critical tenants, reduce oscillations for medium/low priority tenants, avoid global fail-open.

Design summary: - Eviction smoother: exponential moving average over eviction pressure (evict_score) with tunable alpha, plus an eviction budget token bucket to ensure we never starve necessary evictions. The smoother adds hysteresis to node-level decisions so short spikes don't cause mass drains. - Probabilistic fallback: new control-plane decision layer that takes current SLO breach signal, regional capacity state, and route priority to compute a fallback probability p(t). For p < 1 we introduce a randomized short delay before triggering fallback, and we back off probability if the target region is itself at capacity. - Operator knobs: smoothing window, token refill rate, min/max fallback probability per priority class, and emergency override for oncall.

Checklist: - [x] Add smoother implementation in runtime scheduler - [x] Wire smoother into SLO enforcer path - [x] Implement probabilistic fallback heuristic in control plane - [x] Add unit and deterministic integration tests for hysteresis + probability ramping - [x] Add a small ops doc describing knobs and recommended defaults - [x] Hotfix migration plan: default to conservative smoothing (safe by default) and enable rollback via feature flag.

Testing: unit tests for statistical properties (moving average behavior) and deterministic integration tests that simulate node pressure ramps. QA ran 1h soak in staging with synthetic spike load; observed 48% fewer full-node drains and 30% lower rejoin churn. Telemetry updates added to track evict_score, smoother_value, fallback_probability.

Risk and rollback: conservative defaults minimize risk; feature flag gate added at control plane and runtime. Rollback is immediate by toggling flags and a reversion patch is ready.

Related: addresses SUP-1182 and reduces noise observed in ENG-3412.

Priya Menon (reviewer): Can we prove that the smoothing window won't unreasonably delay required evictions for high-priority tenants? Please add a token bucket that guarantees minimal eviction rate.
Marco Alvarez (author): Added eviction token bucket (runtime/scheduler/eviction_smoother.go). Token refill is configurable and defaults to 1 eviction per 30s for critical classes. See added unit tests that verify the bound.
Samir Gupta (reviewer): The probabilistic fallback introduces nondeterminism; ensure tests are deterministic and add telemetry for probability decisions so we can debug in prod.
Marco Alvarez (author): deterministic tests added using fixed RNG seed and a test-only injector. Telemetry counters: fallback_probability_histogram, fallback_decision_sample (route_id, p, result).
Jae Park (reviewer): Suggest tightening default fallback probability ramp for medium routes; current default seemed aggressive in staging runs.
Marco Alvarez (author): Lowered medium priority ramp slope and updated docs/ops recommendations. Re-ran staging soak; improved churn metrics.
Priya Menon (approval): Approving after token-bucket and telemetry additions. Observability looks adequate.
Samir Gupta (approval): LGTM. CI reruns clean.
