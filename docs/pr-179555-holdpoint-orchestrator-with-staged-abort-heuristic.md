# holdpoint-orchestrator-with-staged-abort-heuristics

Motivation: We need a lightweight orchestration primitive to hold traffic at explicit 'holdpoints' during canary/A-B rollouts so that teams can run targeted smoke checks, manual inspection, or additional automated heuristics before allowing further traffic expansion. This PR introduces a Holdpoint Orchestrator that integrates with the existing rollout runtime and adds staged-abort heuristics driven by telemetry windows and configurable embargo windows.

What this changes: Implements a controller and runtime hook that can insert ephemeral holdpoints into a rollout plan. When a holdpoint is hit the orchestrator: 1) pauses traffic expansion, 2) triggers a configured validation pipeline (smoke, sampled replay, lightweight A/B checks), 3) evaluates a configurable set of heuristics (latency drift, error budget burn rate, quality regression signals), and 4) either advances, keeps paused, or triggers automated staged-abort with graceful rollback steps.

Key features: - Declarative holdpoint objects as part of rollout policies (YAML examples included). - Pluggable heuristics registry: teams can register custom evaluators; built-in heuristics include latency-percentile-delta, error-rate-slope, embedding-consistency-signal. - Embargo windows: ability to prevent automated aborts during scheduled maintenance or regulated blackout windows. - Session affinity preservation: when a rollback happens we ensure session-coherent swap by leveraging the existing session-lease mechanism. - Dry-run shadow validation: optional dry shadowing mode to run validations without affecting control plane metrics. - Telemetry anchoring: validations are anchored to a telemetry window with a lookback and stale-data guard.

Why this is different: Unlike the existing heavy-weight canary controllers, Holdpoint Orchestrator is intentionally small-surface and composable; it focuses on stopping progress at predictable checkpoints and running domain-specific guards rather than trying to be fully autonomous. This is targeted at product teams that want human-in-the-loop safety combined with reproducible automatic heuristics for routine regressions.

Breaking down the changes (high level): - rollout/holdpoint/controller.go: new controller for creating/observing holdpoints and orchestrating validation runs. - rollout/engine/staged_abort.go: staged abort state machine and graceful rollback planner. - rollout/policy/holdpoint_schema.yaml: policy schema and examples. - telemetry/validators/latency_delta.go, error_slope.go, embedding_consistency.go: built-in heuristic evaluators. - e2e/holdpoint_integration.yaml: new integration test harness that runs a mini A/B sequence with a holdpoint. - docs/rollouts/holdpoints.md: docs and examples plus troubleshooting guide.

Testing and rollout plan: - Unit tests added for controller state transitions and evaluator contracts. - Integration job added: e2e/holdpoint_integration.yaml (runs in the canary lab). - We ran a staged deploy to the internal dev cluster using a dry-run holdpoint and exercised the latency_delta and error_slope heuristics. Observed correct pause/advance behavior and session-affinity preservation during swap.

Metrics and observability: - New metrics: rollout_holdpoint_duration_seconds, holdpoint_validation_latency_seconds, holdpoint_abort_reason_total (labels: evaluator, rollout_id). - Traces emitted for each validation run and decision path. Dashboards updated with a Holdpoint view (docs include Grafana panel snippets).

Operational concerns: - Default config keeps automated aborts conservative (requires 2 heuristics to trip) to avoid noisy rollbacks. - Admins can define embargo windows using cron-like schedule strings in the rollout policy. - Rollback is staged: first attempt soft-fallback to a compatible model variant, then a full atomic swap revert if soft-fallback fails; both steps are instrumented and reversible.

Migration notes: Existing rollouts are unaffected unless a holdpoint is declared. The new schema is additive.

Checklist: - [x] Controller and runtime hook implemented - [x] Built-in evaluators implemented - [x] Unit + integration tests - [x] Docs and examples - [x] Telemetry and dashboards updated

Related: ENG-3421 (holdpoint design), ENG-3499 (telemetry anchoring), SUP-1123 (customer incident motivating embargo windows).
Marcus Lee: If a holdpoint sits indefinitely because telemetry is delayed, do we have a stale-data escape hatch?
Priya Kapoor (reply): Yes — controller enforces a max_holdpoint_ttl and emits stale_holdpoint_alert. TTL default is 30m; adjustable in policy.
Elena Gomez: Can embargo windows be inherited across nested rollout groups?
Priya Kapoor (reply): Currently not inherited automatically; we opted for explicitness. Open to follow-up if teams want inheritance — linked ENG-3580 for that work.
Daniel Park: Request small perf benchmark proving low scheduling overhead (added benchmarks in e2e harness).
Priya Kapoor (reply): Added microbenchmarks and included summary in PR description (see additions).
Marcus Lee: Approve after addressing docs clarity around soft-fallback semantics.
Elena Gomez: Approved.
