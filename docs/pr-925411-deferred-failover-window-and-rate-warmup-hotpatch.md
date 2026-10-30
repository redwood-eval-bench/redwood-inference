# deferred-failover-window-and-rate-warmup-hotpatch

Context: During a 2026-02-09 incident we observed aggressive cross-region failover decisions caused by transient KV-cache pressure and short-lived CPU spikes. Immediate failover was triggering cascading load to secondary regions and exacerbating latency SLO violations. This PR introduces two coordinated mitigations: 1) a deferred failover window (soft holdoff) that delays irreversible cross-region failover decisions for a small, configurable window to let transient errors recover; and 2) a per-tenant rate warmup queue that staggers new traffic ramps after failover or node additions to avoid cold-ramp bursts. Motivation: Reduce noisy failovers and downstream amplification that caused higher 99p latency and tenant evictions, without removing the ability to failover for genuine sustained outages.

What this changes: - Introduces failover_controller.deferredWindowMs and a state machine transition path that moves a region into a "pending-failover" state before executing a hard failover. If health metrics (latency, error-rate, KV cache pressure) recover within the window, the controller cancels the failover and records the recovery reason. - Adds a tenant-level QueueWarmup with token-bucket backed gradual admission; new tenant/route traffic is admitted at a configured ramp-rate for the first N seconds after a topology change. - Adds telemetry: failover.window_started, failover.window_cancelled, warmup.admitted_tokens, warmup-throttled-tenants. These feed the existing Grafana dashboards and alerting rules are adjusted to avoid noisy alerts during the window. - Tight coupling between the two features: when a pending-failover is active, warmup ramp-ups for tenants routed to the impacted region are applied more conservatively to avoid overloading fallback regions.

Testing and validation: - Unit tests for state transitions and queue warmup behavior (failover_controller_test.go, queue_manager_test.go). - Integration tests simulating transient KV pressure with deterministic clock advance to verify window cancellation and metrics emission. - Load test in benchmark-lab mimicking a cold failover: results show 30% reduction in spike QPS to fallback region and 22% improvement in 99p latency during the recovery window. - Local fault injection runbook and chaos experiment notes included.

Rollout plan: - Canary on dedicated us-west-2 region (10% of cross-region traffic) for 48 hours, with tight SLO watch and auto-rollback if 99p latency increases >5% vs baseline. - After canary, staged rollout across regions with slow warmup ramps and ops watch. - Ops playbook updated with steps to override the deferred window (immediate failover) in case of sustained outage.

Checklist: - [x] Implementation code and tests
- [x] Telemetry and alert tuning
- [x] Benchmark results attached
- [x] Verified canary plan
- [x] Docs PR (link in comments)

Related: Linear ENG-3892 (incident follow-up), ENG-3918 (failover hardening). Jira SUP-2147 contains the customer escalation that motivated a fast patch.
Marco Li: Left a few suggestions around naming and ensuring the pending-failover state is observable. Can we add a metric with the pending reason?
Aisha Malik: Added pending_reason label to failover.window_started and emit pending_reason on cancellation. Updated tests to assert label presence.
Priya Desai: Can we make the default window conservative (250ms) and expose via feature-flag? Concerned about long windows increasing user-facing error tolerance.
Aisha Malik: Default set to 250ms, feature-flag controlled per-region. Ops can bump via config for region-specific tuning.
Evan Thompson: Integration test flaky on CI (timing). Suggested using a deterministic clock rather than sleeps.
Aisha Malik: Reworked integration test to use fake clock. Re-ran CI and integration stabilized.
