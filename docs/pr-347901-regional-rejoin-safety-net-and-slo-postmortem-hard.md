# regional-rejoin-safety-net-and-slo-postmortem-hardening

Summary: This PR implements a targeted safety-net for regional rejoin events and hardens SLO enforcement after the recent incident where aggressive rejoin attempts caused repeated KV bake/replay storms. The changeset is motivated by the postmortem for INC-2026-02-03 (Linear ENG-1972): nodes in an islanded region repeatedly flipped circuits while attempting to rejoin, amplifying eviction churn and pushing SLO windows over thresholds.

What this PR does (single-string narrative): Adds a 'rejoin-safety' controller that enforces backoff and health-score gating for rejoin attempts, surfaces a lightweight 'circuit-salvage' shim that prefers conserving sticky KV leases when safe, and extends the SLO audit trail so that the scheduler can record and surface the decision tree that led to a shed or allow action. Also includes telemetry anchors (trace tags and a new slo_reason dimension) so SLO-driven decisions can be aggregated in dashboards without additional instrumentation.

Motivation and context: During a regional flapping event the rejoin loop triggered rapid KV cache repopulation, which in turn caused resource pressure on peers and led to cascading soft-failovers. Our existing circuit breaker prioritized availability but lacked an explicit guarded rejoin path; this PR introduces a safety net that stalls rejoin until probes confirm cross-region KV coherence and until SLO-leeway budgets are checked. The approach prioritizes graceful degradation: prefer limited, predictable capacity with a stable KV prefix over aggressive rejoin that creates noisy oscillations.

Design notes: - Rejoin controller: state machine (PENDING -> PROBING -> PERMITTED/DEFERRED). Probing uses lightweight heartbeats with KV consistency checks (max 50ms added latency). Permitting is rate-limited by a token-bucket that is charged from a new 'slo_autopsy' budget. - Circuit salvage shim: when a node meets specific criteria (low inbound load, KV prefix intact, and a clean probe), we attempt a sticky-lease-preserve path instead of full rebuild. This reduces token and network spike. - Telemetry: add slo_reason tag (values: "rejoin_probe_fail", "slo_budget_exhausted", "salvage_ok", "salvage_denied"). Emit debug traces for the first 5 rejoin attempts per incident to keep logs actionable.

Files changed (high level): - scheduler/rejoin_controller.go: main state machine and gating logic. - runtime/circuit_salvage.go: salvage heuristics and sticky-lease logic. - telemetry/tags.go: new slo_reason dimension and tagging helpers. - dashboard/queries/slo_anchors.sql: prebuilt query for SLO rollup. - integrations/k8s/helm-values.yaml: small default tuning for rejoin window. - tests/integration/rejoin_harness_test.go: deterministic harness for flapping scenarios. - ci/flakeguard.yml: add a job to catch rejoin-related flakiness.

Commit summary: - 9ea5b3f: add rejoin controller skeleton and state transitions. - a3f42d1: implement probing and token-bucket SLO budget check. - b2c9130: circuit salvage heuristics and sticky-lease preserve path. - c5d7f88: add telemetry tags and dashboard query. - ffa1b2d: integration harness and unit tests. - 11de77a: docs and helm defaults.

Testing and validation: Local integration harness simulates regional partition and measured outcomes: previous baseline (no safety net) produced 3x rejoin attempts in 60s and SLO breaches at p95; with this changeset rejoin attempts limited to 1-2 per minute per region and SLO breach windows reduced to safe margins. Added unit tests for state transitions and property-based checks for token-bucket accounting. CI ran full integration suite and flakeguard; one flaky test in an unrelated module was quarantined during this run and re-run.

Rollout plan: Canary on staging for 24 hours with 10% of regional traffic routed through the new rejoin controller path (feature flag rejoin.safety_net=v1). Monitor slo_reason metrics and slo_anchors dashboard for regressions. If telemetry shows unanticipated stalls, revert by toggling rejoin.safety_net to off; rollback playbook attached in the postmortem doc (Linear ENG-1972).

Backwards compatibility & operator impact: No user-visible API or model changes. Operators should be aware of new helm tunable values (rejoin.window_seconds, rejoin.token_rate) and the new telemetry slo_reason dimension. No breaking changes.

Related work: Addresses ENG-1972 (incident postmortem), ties into SRE-81 for longer-term SLO-budgeting work. This PR is intentionally scoped and includes a hotfix pathway that can be backported to release/2026-02 for immediate mitigation.

Checklist: - [x] Unit tests for controller transitions - [x] Integration harness for flapping scenario - [x] Telemetry anchors and dashboards updated - [x] Runbook updated and linked - [x] Approved by SRE and runtime owners

Notes on scope and future work: This is a tactical safety-net; follow-on work will refactor the token-bucket SLO accounting to use the global SLO allocator (ENG-2034) and expand salvage heuristics to support cross-zone sticky leases.

Diego Ramos: Left a few nits on gating thresholds; suggested reducing default token rate to 0.5/min for canary.
Aisha Patel: Incorporated Diego's suggestion, added helm tunable and documented canary guidance.
Lena Cho: Approved after confirming telemetry tags and dashboards. Asked for explicit guard to avoid deadlock if probe subsystem fails.
Aisha Patel: Added probe-failure fallback (defer->manual-intervention after 15 minutes) and updated runbook.
Marcus Lee: Ran canary in staging and signed off based on reduced rejoin oscillation metrics. Approved merge.
