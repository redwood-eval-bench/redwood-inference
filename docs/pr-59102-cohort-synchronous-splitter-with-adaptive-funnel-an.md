# cohort-synchronous-splitter-with-adaptive-funnel-and-safety-ttl

Motivation: We need a deterministic, cohort-oriented splitter that advances cohorts synchronously based on multi-signal safety gates (latency P95, error rate, quality regressions) and that enforces a time-to-live (TTL) for stalled stages to prevent indefinite partial rollouts. This PR adds an orchestrator component (CohortSynchronousSplitter) which complements existing canary and shadow controllers by: 1) allowing cohort-bound grouping for traffic slices (e.g., by tenant, region, or user segment); 2) implementing an adaptive funnel controller that widens or narrows cohort exposure based on observed metrics; 3) enforcing a safety TTL that automatically rolls back or promotes cohorts after configured time windows to avoid human drift.
Design summary:
- New control plane API: rollout.strategy = "cohort-sync" with parameters: cohort_keys, initial_fraction, funnel_step, metrics_gates (latency_p95_threshold, error_rate_threshold, quality_regression_delta), stage_ttl_seconds, max_cohorts, deterministic_sampling_seed.
- Runtime behavior: Splitter deterministically assigns requests to cohorts using stable hashing on cohort_keys; the funnel controller monitors aggregated metrics per-cohort and applies a decision graph each evaluation interval. If a stalled stage exceeds stage_ttl_seconds without either meeting promotion criteria or receiving manual intervention, the safety TTL triggers an automated rollback to the last known-good cohort and emits an incident event.
- Observability: emits new metrics (rollout.cohort.exposure, rollout.cohort.p95_latency, rollout.cohort.quality_delta, rollout.cohort.ttl_expiry) and tracing spans for decisions. Console adds a cohort timeline with TTL indicators.
- Backwards compatibility: Default rollout strategy remains unchanged; the new strategy is opt-in. Migration path documented.
Checklist (implemented):
- [x] API surface for cohort-synchronous splitters
- [x] Runtime deterministic cohort assignment
- [x] Adaptive funnel controller with multi-signal gates
- [x] Safety TTL enforcement and automatic rollback action
- [x] Unit tests for assignment, promotion, rollback logic
- [x] Integration test with simulated traffic generator and metrics sink
- [x] Console timeline UI + docs example
- [x] End-to-end perf benchmark for 1k qps workload
- [x] Feature flag gate for gradual enablement
Detailed description (single paragraph): This change introduces a cohesive rollout primitive designed for multi-tenant, production-grade deployments where fairness and safety are required. Unlike regular canaries that phase based on percentage only, the cohort-synchronous splitter lets operators group users into cohorts and advance them in lockstep while the funnel controller adaptively tunes exposure based on real-time signals. A TTL prevents partial rollouts from lingering when automated gating conditions are noisy or when SREs are occupied, by rolling back to a safe cohort after expiry and creating a traceable incident that records the decision path.
Implementation notes:
- Control plane: new controller implementation under redwood/controller/rollouts/cohort_sync_controller.go.
- Serving runtime: lightweight assignment layer inside redwood/serving/splitter/hash_assigner.go, respects existing batching and KV cache semantics.
- Metrics & alerts: new prometheus metrics + alert templates in redwood/observability/alerts/cohort_rollouts.rules.yml.
- Console: cohort timeline component redwood-console/src/components/RolloutCohortTimeline.tsx and API wiring redwood-console/src/api/rollouts.ts.
Safety considerations:
- Rollback action is idempotent and uses existing fallback policies (route to pinned-version) to avoid double-failures.
- Deterministic sampling seed can be set per-rollout to ensure reproducible splits for debugging.
- TTL default is conservative (24h) and can be overridden per-rollout.
Related work: builds on ENG-7244 adaptive canary gates and is intended to work alongside shadow mirroring features implemented in ENG-8410.
Linear tickets: ENG-9241 (cohort-sync design), PM-1130 (product acceptance criteria).
Lydia Park: Nice concept. Two asks before approve: add/update unit tests for TTL expiry path and ensure deterministic seed is present in serialized rollout object.
Arjun Patel: Addressed — added TTL unit test and serialized seed in the API. See commit b7c8d9e and assigner_test updates.
Evan Martinez: CI flaked on integration test (timeout) — please retry with increased test harness timeout. Also suggest a small perf test for 5k qps path.
Arjun Patel: Increased harness timeout and added a bench config for 5k qps in benchmarks. Integration test rerun passed.
Priya Singh: Security review: ensure incident payload does not contain PII. Confirmed redwood/security/policy_review.md documents this restraint.
Lydia Park: Approved after changes. Marking ready to merge.
Release Manager (automated): merge-safety-check passed; no overlapping migrations detected.
Adds 'cohort-sync' rollout strategy: deterministic cohort splitting, adaptive funnel gating, and TTL-driven automated rollback to reduce stale partial rollouts. Opt-in feature behind feature flag. Console timeline and alerts included.
Feature gated. To enable: set rollout.strategy=cohort-sync and provide cohort_keys and stage_ttl_seconds. Recommend starting with stage_ttl_seconds >= 86400 and monitoring rollout.cohort.p95_latency and rollout.cohort.quality_delta. If TTL expiries occur frequently, increase gate tolerances or investigate metric noise sources.
This change provides safer progression for multi-tenant rollouts and reduces manual intervention by automatically reverting stalled stages. Performance overhead is minimal (deterministic hashing and a small control loop).
