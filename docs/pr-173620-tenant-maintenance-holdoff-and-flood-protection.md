# Tenant maintenance holdoff and flood protection for circuit maintenance mode

Motivation: During recent maintenance windows and rolling kernel upgrades we observed tenant-level eviction storms triggered by simultaneous circuit transitions and aggressive prioritizer heuristics. This caused fragile tenants to be repeatedly evicted, increasing tail-latency and recovery time. This PR introduces a 'maintenance holdoff' that temporarily softens eviction aggression for tenants that have been recently moved into maintenance mode, plus two complementary flood-protection measures (rate-bounding for circuit transitions and a short-lived tenant grace lease) to prevent cascade failures.

What this changes: 1) Maintenance holdoff: when an operator marks a node or shard as 'maintenance', affected tenants enter a 30s holdoff during which the prioritizer reduces eviction weights and avoids hard preemption. 2) Transition rate-bounding: a cluster-wide leaky-bucket limits circuit transition events to a configurable burst (default 5 transitions / 10s) to avoid simultaneous rebalances. 3) Tenant grace lease: a short-lived lease (default 45s) allows in-flight requests to finish without being forcibly evicted; the lease is observable and audited. 4) Telemetry: new metrics and debug traces for holdoff starts/ends, rate-bucket fills, and lease expiration reasons. 5) Rollout-safe: feature gated behind runtime flag "maintenance_holdoff_v1" with default enabled for safety team; can be toggled per-region.

Implementation notes: changes touch the prioritizer, circuit orchestrator, operator CLI, and observability hooks. Logic added in prioritizer/holdoff.go, orchestrator/rate_limiter.go, operator/commands.go, and telemetry/metrics_registry.go. Tests: unit tests for holdoff state machine and leaky bucket; integration test that simulates a rolling maintenance with 100 tenants to validate no mass-evictions.

Checklist: - [x] Design doc linked (Linear ENG-1987) - [x] Unit tests added - [x] Integration test added - [x] Metrics and traces wired - [x] Runtime flag and CLI hook - [x] Docs blurb for maintenance runbook (internal) - [x] Observability dashboard tile prototype

Rollout plan: 1) Canary on non-prod regions (2/20) 2) Monitor new metrics for two maintenance cycles 3) Gradual enable for production regions 4) Enable cluster-wide by config after SRE signoff. Rollback: runtime flag off or change leaky-bucket thresholds; revert is safe since state machine tolerates stale holdoff entries.

Related work: touches codepaths from ENG-1223 (tenant prioritizer refactor) and ENG-1544 (operator CLI improvements). Support ticket SUP-9142 reported repeated evictions during a kernel upgrade and motivated the flood-protection rate-bucket.

Notes on safety: The holdoff only reduces eviction aggressiveness; it never disables critical safety evictions (e.g., overloaded GPU OOM conditions). We included a safety bypass when node health drops below critical thresholds.

Commits summary: - c1a7d9b: add maintenance holdoff state machine and tests - 0f3e2a1: implement cluster-wide leaky-bucket rate limiter for circuit transitions - 9b4c8f2: add tenant grace lease and audit events - 7d0f11e: wire metrics and add dashboard tile stub - a2b3c4d: add operator flag and CLI integration - 4e5f6a7: integration test for rolling maintenance scenario and flake fixes

Changed files (high-level): - prioritizer/holdoff.go (new) - orchestrator/rate_limiter.go (new) - orchestrator/transition_manager.go (updated) - operator/commands.go (updated) - telemetry/metrics_registry.go (updated) - tests/unit/holdoff_test.go (new) - tests/integration/maintenance_rollout_test.go (new) - docs/internal/maintenance-runbook.md (updated) - dashboards/maintenance_holdoff.json (new) - ci/config.yml (tweaked to add integration stage) - pkg/lease/grace_lease.go (new)

CI and checks: initial CI run failed on integration stage (transient race in test harness: TestMaintenanceRollout flake). I reproduced locally and added deterministic timing controls in integration harness (commit a2b3c4d). Final CI run: unit tests pass, integration suite pass, lint pass, e2e smoke pass. Post-merge: we will keep an eye on integration flakiness metric.

Review thread (high-level): see review comments below. The PR was iterated twice to address concerns about tenant fairness scoring and to add additional observability for lease expirations. After addressing those comments the change received approvals and was squashed and merged.

Merge decision: Squash-and-merge after CI green and SRE approval. Merged by Lina Chen on 2026-02-20.
Stability: Add tenant maintenance holdoff window and flood-protection heuristics to circuit maintenance mode; reduces churn during multi-tenant node maintenance and prevents tenant-level eviction storms.
Samir Gupta (2026-02-16): Can we ensure the holdoff doesn't bias tenant fairness permanently? Suggest adding a decay so weights return to normal after 2 minutes. - Response: added decay in holdoff.go and unit test (commit c1a7d9b).
Lina Chen (2026-02-17): We need better observability for lease expirations (reason codes). Please add audit events and a metric. - Response: added audit events and metrics (commit 9b4c8f2).
Diego Ruiz (2026-02-18): Integration test flaked locally; reproducible race in test harness. Add deterministic timers or mocks. - Response: updated integration harness and made timers deterministic (commit 4e5f6a7).
Samir Gupta (2026-02-19): Approved after verifying integration run on canary region; recommend watching 'maintenance_holdoff.evictions_avoided' metric for first 48h. - Response: noted in rollout plan.
