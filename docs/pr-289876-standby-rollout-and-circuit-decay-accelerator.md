# standby-rollout-and-circuit-decay-accelerator

Motivation: In previous incidents we observed tenant circuits getting stuck CLOSED for long periods during regional failovers and noisy neighbors would prevent healthy tenants from reclaiming capacity. This PR introduces a coordinated standby rollout strategy plus a 'circuit decay accelerator' to (1) reduce blast radius during regional failover, (2) speed recovery of false-positive circuit closures, and (3) provide an operator knob for staged re-admittance of tenants.

What this PR does (single-string sectioned summary): Adds a new StandbyRollout controller in services/slo-enforcer that stages tenant traffic into a standby pool before full rejoin; implements a bounded exponential decay function (CircuitDecayAccelerator) in runtime/circuit that increases the circuit reopen rate when telemetry indicates healthy recovery; introduces a new reconciliation API (internal) for manual operator-triggered fast-unstick; expands telemetry to emit circuit_transition_reason and decay_score metrics; wires a feature-flag to enable gradual rollout per region.

Key design decisions: Standby pool concept keeps a small fraction (configurable default 5%) of tenant slots reserved as warm standby to absorb bursty rejoin traffic. The decay accelerator uses a conservative capped exponential backoff with jitter to avoid thundering reopen storms: circuits progress from CLOSED -> PROBATION -> OPEN in bounded steps, where PROBATION admits a limited token bucket for sampled requests to validate safety. Operator controls include: decay_multiplier, probation_window_secs, standby_capacity_pct, and emergency_bypass_token.

Files changed (high-level): Implementations in Go for the controller and API, small runtime C++/Rust shim to expose decay hooks to the hot path, protobuf additions for the new reconcile endpoint, and dashboards/alerts in observability-pack. No public API changes.

Testing and rollout: Unit tests for controller logic and fuzz tests for decay function added. Integration tests run against the perf-canary harness with synthetic tenant spikes; results show a 35% reduction in long-lived CLOSED circuits under simulated noisy-neighbor failovers and a median recovery time improvement from ~12m to ~4m for false-positive closures. Rollout plan: enable feature-flag in canary region, monitor decay_score and circuit_transition_reason, then flip per-region flags over 48h windows.

Operational playbook (short): 1) If a region shows elevated steady CLOSED count, enable emergency_bypass_token to drain probation queues. 2) Use reconcile API to fast-unstick individual tenant circuits (documented endpoint path). 3) If token-bucket admission causes unexpected latency, reduce standby_capacity_pct or probation_window_secs.

Risk and mitigations: The main risk is reopening too many circuits concurrently and causing downstream overload; mitigations are conservative caps, token-bucket admissions, and automatic fallback to previous circuit policy if latencies breach configured SLOs. Also added extra telemetry and e2e canary checks before global rollout.

Related work: Complements ENG-3910 (regional probe stabilizer) and avoids duplication with recent circuit snapshot work. Linked SUP-1147 where customer-reported stuck circuits motivated this patch.

Summary: Provides a pragmatic, operator-friendly mechanism to stage rejoin traffic and accelerate safe circuit recovery while preserving SLOs and preventing mass reopen storms.
Diego Martinez: Please add a short-circuit to disable decay accelerator under emergency 'do-not-open' mode.
Aisha: Added emergency_bypass_token and a global disable flag in featureflags (see handler.go).
Hannah Liu: Can you add more unit coverage around jitter math (edge case when multiplier->1.0)?
Aisha: Added fuzz tests and a targeted unit for jitter edge case (fuzz harness entry).
Omar Khouri: Observability looks good; add a composite alert for steady CLOSED count + low decay_score.
Aisha: Composite alert added to dashboards and alert rules in observability-pack.
Introduces standby rollout staging and a circuit decay accelerator to speed safe recovery from CLOSED circuits; adds operator reconcile API and new telemetry (circuit_transition_reason, decay_score). Feature-flagged by default; rollout via canary regions.
