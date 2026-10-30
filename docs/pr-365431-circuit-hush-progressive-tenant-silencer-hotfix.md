# circuit-hush-and-progressive-tenant-silencer-hotfix

Motivation: During a recent regional overload incident we observed noisy circuit flips and aggressive tenant quarantines that caused cascading KV cache drains and prolonged regional failovers. This patch introduces a temporary 'circuit hush' mechanism to suppress noisy circuits for a short dampening window and a 'progressive tenant silencer' algorithm that applies graded traffic reductions instead of immediate hard quarantines.

What this PR does:
- Add circuit hush state to the circuit manager with a configurable hush_window and dampening_factor so circuits that flip rapidly enter a suppressed state for N seconds before normal evaluations resume.
- Implement progressive tenant silencer in the SLO enforcer: instead of immediate full quarantine when SLO breach thresholds are hit, tenants receive stepwise traffic reductions (80% -> 50% -> 20%) with backoff timers and auto-reinstate on metric recovery.
- Improve regional failover compatibility: ensure standby routing preserves per-tenant KV cache affinity when switching to fallback regions to avoid cold startups.
- Emit new telemetry: slo_violation_score, tenant_silence_level, circuit_hush_count, hush_duration_ms for better incident diagnosis and to power future automated remediation.
- Hotfix a bug where the previous standby swap path dropped kv_preserve flag, causing cache thrash during failovers.

Compatibility notes:
- Default behavior is off for hush and progressive silencer; enabled via runtime flags and defaults.yaml entries added in this PR. Customers using custom rollout policies should verify their existing canary rules since tenant-silencer adds a new weighting factor to admission decisions.

Testing and rollout plan:
- Unit tests for circuit manager hush transition and tenant silencer state machine added.
- Integration test simulating region overload added (tests/slo_integration_test.go) that asserts KV retention and gradual traffic reduction.
- Canary rollout: enable in a single non-critical region behind a feature flag, monitor slo_violation_score, and circuit_hush_count metrics for 48h before wider rollout.

Checklist:
- [x] Add config knobs and defaults
- [x] Add metrics and traces
- [x] Unit + integration tests
- [x] Docs update stub (docs/alerts.md)
- [x] Hotfix to standby routing kv_preserve flag

Related: ENG-4042 (incident analysis), ENG-4120 (slo-enforcer backlog), SUP-9981 (customer support case referencing prolonged failover), design doc: /docs/design/tenant-silencer.md
Mateo Ruiz: Can we add an explicit metric that counts how many tenants are in each silence level? It will help SRE to quickly gauge impact. (resolved: added tenant_silence_level)
Clara Liu: The hush window feels like it could mask real regressions if set too high. Recommend a conservative default (10s) and a ramping plan. (author reply: default set to 10s, documented rollout plan)
Ethan Park: Tests are solid but CI flaked once on the integration test due to timing; I suggest increasing the test timeout by 30% to reduce flakes. (author fixed test timeout and re-ran CI)
