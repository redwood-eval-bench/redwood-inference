# Traffic Lantern: safety controller for granular model cutovers and session-consistent shadowing

Adds a new rollout safety controller (Traffic Lantern) that enables fine-grained holdpoints, session-consistent shadow queues, and deterministic rewind-playback for safe cutovers. No API breaking changes; opt-in via rollout policy config.
Motivation: We need a compact, opt-in safety controller for model cutovers that operates at slice and session granularity. Traffic Lantern provides holdpoints, a lightweight session-consistent shadow queue for fidelity checks, and deterministic rewind-playback to reproduce and revert suspect slices.
Design summary: Traffic Lantern is implemented as a policy plugin to the rolls controller; it maintains an in-memory heatmap of slice acceptance rates, exposes a safety budget per rollout, and gates traffic by canonical holdpoints (pre-commit, smoke, canary, promote). The plugin integrates with the existing telemetry pipeline and emits metric anchors used by the rollout conductor.
Changes include: 1) controller/lantern: new policy implementation and config schema; 2) runtime/router: small adapter to tag session ids and mirror to shadow queue; 3) telemetry: new metric labels and a debug trace endpoint; 4) e2e and unit tests.
Checklist: - [x] Design doc linked (Linear ENG-2482) - [x] Policy schema added with default-safe values - [x] Backward compatible: Lantern is opt-in via rollout.policy.type - [x] Added e2e smoke that replays a failed canary and validates rewind-playback - [x] CI green (after a flaky test fix).
Testing & validation: Local fuzzed traffic tests, CI integration tests with synthetic heatmaps, and a staged demo in the perf-canary infra.
Rollout plan: feature flagged and staged to canaries for 48h, promote only if rollback-rate < 0.5% and latency P95 delta < 10ms for two consecutive windows.
Migration notes: Operators must add a \"lantern\" policy block to rollout specs; otherwise behavior unchanged.
Related work: builds on shadow-bridge plumbing in controller and reuses the session-lease code in redwood-router.
Security: no new network surfaces; shadow queue is internal-only and subject to existing RBAC.
TODOs: follow-up for UI visualization (Console ticket ENG-2499) and optional persisted heatmaps for long-running rollouts.
Julian Park: Nice compact design. Two asks: clarify memory bounds for the heatmap and add a short-circuit when safety budget is exhausted.
Author (Ava): Added memoryBound config and early-fail path in lantern.go; updated description and tests to cover exhaustion case.
Lina Zhang: Can we ensure mirrored shadow traffic respects sampling and does not double-count quota? Please add quota-aware enqueueing.
Author: Router adapter updated to respect sampling and a feature-flag gate; added unit tests for quota-aware enqueueing.
Rohan Gupta: e2e looks good; would like a clearer metric name for the rewind-playback event so SLO alerts can reference it.
Author: Renamed metric to rollout_rewind_playback_total and updated telemetry docs.
CI: initial run failed on flaky integ test (timeout). Subsequent run passed. Approval set by Julian and Lina.
Julian Park: Approve after fixes.
Lina Zhang: Approve.
