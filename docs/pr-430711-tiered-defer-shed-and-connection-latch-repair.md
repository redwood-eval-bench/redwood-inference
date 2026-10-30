# Tiered defer-shed and connection latch repair

Motivation: We observed a class of incidents where short-lived regional egress spikes combined with aggressive connection reopen logic caused repeated tenant circuit flips and unnecessary evictions. This PR introduces a conservative, tiered 'defer-shed' mechanism that delays hard egress shedding for recently warmed connections and adds a connection latch that prevents rapid open/close cycles from retriggering circuit state transitions. Changes include runtime logic, new telemetry, tests, and a small CLI to toggle the defer tier for canaries. Checklist: 1) Add tiered defer-shed policy and config knobs; 2) Implement connection latch with epoch-backed cooldown; 3) Wire telemetry for shed/latched events and expose two histograms; 4) Add unit and integration tests; 5) Run directed load tests and smoke on staging; 6) Prepare release notes and roll-out plan. Impact: aims to reduce churn in regional failovers and avoid cascading tenant evictions during transient hotspots. Rollout: staged canary (5% -> 25% -> 100%) with metric thresholds and automatic rollback on increased eviction rate. Related: ENG-1987 tracks follow-ups for per-tenant tuning and Console flags.
Diego Rivera: Found a race in latch expiry when node clocks skewer; suggest switching to monotonic epoch counters rather than wall time.
Maya Chen: Switched to epoch counter backed by leader-seq (see latch.go changes). Added a unit test that simulates clock drift.
Anjali Patel: Can we ensure the deferred-shed path emits the same tracing spans as normal shed? Ops need consistent traces.
Maya Chen: Added trace forwarding in egress_controller.go; defer path now annotates span with 'defer_shed=true' and logs the tier name.
Noah Kim: CI flaky test in integration suite (TestDeferShedTransientSpike) failed once during first run, flake root cause looks like test timing; re-ran and passed.
Anjali Patel: Approve after minor nits fixed around log levels and one comment on histogram buckets.
Diego Rivera: LGTM, will squash and merge when staging metrics look good at 25% canary.
Maya Chen: Deployed 5% canary to staging noon 2026-02-15; observed 12% reduction in eviction churn and no increased latency percentiles. Proceeding to 25% on 2026-02-16.
Introduce 'defer-shed' tier that delays hard egress shedding for recently warmed connections to reduce tenant eviction churn during transient spikes.
Add connection latch to prevent rapid open/close cycles from retriggering circuit transitions and causing oscillation.
New metrics: shed_deferred_total, connection_latched_duration (histogram).
Config knobs: runtime.defer_shed.enabled, runtime.defer_shed.cooldown_ms, runtime.connection_latch.window_epochs. Rollout controlled via canary_ctl.
No breaking changes expected. Backfill: operators should monitor eviction rate and shed_deferred_total during rollout.
