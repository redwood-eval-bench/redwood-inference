# Introduce invocation summaries and batch-metric-optimizer for quieter alerts

Context and motivation: Customer-facing incidents in January showed a lot of noisy SLO alerts driven by long-tail invocation variance and extremely high cardinality of metric labels (prompt_id, model_revision, region). This PR introduces two cooperating features: 1) invocation summaries: a compact per-invocation rollup that attaches a small set of canonical attributes (invocation fingerprint, cohort, token_count_bucket, model_variant) and consolidated phase timing; 2) batch-metric-optimizer: an offline/nearline pipeline that collapses high-cardinality metrics into cohort-backed aggregates and automatically suggests alert quieting knobs.

What this PR does (high level):
- Adds a small runtime hook that emits InvocationSummaries to the observability pipeline (compact protobuf-like json, max 1kB). Summaries capture phase-level timings, token counts, cache hit flags, and a generated invocation_cohort tag.
- Introduces a batch-metric-optimizer service (k8s deployment) that consumes recent InvocationSummaries and produces: cohort aggregation rules, per-alert quieting suggestions, and a reconciled metrics export that replaces combinatorial labels with cohort keys.
- Adds Grafana dashboard panels and an alert playbook that uses cohort-weighted thresholds and a quiet-window derived from median tail latency per-cohort.
- Provides a compatibility shim so customers that rely on per-token labels see no functional regression; the optimizer exports both raw and cohorted views until customers opt-in.

Checklist (tested):
- [x] Unit tests for summary serialization and validator
- [x] Integration test that verifies cohort assignment and aggregator correctness (small sample)
- [x] End-to-end smoke: summaries -> optimizer -> metrics export -> dashboard renders
- [x] Load test: emitter rate at 5000 invocations/s with <3% extra CPU on the metrics forwarder
- [x] Security review: PRD for minimal PII (no prompt content emitted)

Commits summary: This branch was developed across multiple commits (see commits field). Major logical commits include: "invocation summaries: serializer + emitter", "batch optimizer: aggregator core and rule exporter", "dashboard: cohort panels + runbook links", "compat shim and opt-in toggle", "tests and load profile adjustments".

Files changed (high level): see files_changed list for exact paths. Notable additions: src/observability/invocation_summary_emitter.py, services/batch_metric_optimizer/main.py, dashboards/cohort_rollup_panels.json, docs/observability/invocation-rollups.md, tests/test_invocation_summary.py.

Review conversation (highlights):
- Aaron Liu (reviewer): "Can we guarantee the cohort assignment won't drift during a rollout? If cohorts change semantics we will mute important signals."
  - Author reply: "We implemented stable cohort hashing based on (model_family, sequence_length_bin, cache_miss_flag) and included a cohort-translation table for rollbacks. Added test to assert stable mapping across restarts."
- Marisol Vega: "The optimizer could accidentally remove labels that customers need for billing. Please ensure the export keeps raw metrics behind a feature flag."
  - Author reply: "Agreed. The optimizer writes cohorted metrics to a separate metric namespace (redwood.cohorted.*) and does not overwrite redwood.raw.* until the customer toggles opt-in. Added docs and a console toggle."
- Tom Bennett: "Dashboard looks useful but we should add a synthetic SLO replay to validate suggested quiet windows."
  - Author reply: "Included a small synthetic replay harness under services/batch_metric_optimizer/replay that replays sampled InvocationSummaries and scores quiet-window proposals. Added to CI smoke step."

CI and rollout notes:
- CI: unit and integration tests passed in the last run (flaky network test retried once). Build artifacts pushed to our internal registry.
- Canary rollout plan: 10% of traffic to opt-in cohorting + optimizer suggestions for 48 hours; if no alert lift and no missed SLOs, expand to 50% then global opt-in after manual approval.

Merge outcome: Merged via squash after approvals from Aaron and Marisol. Post-merge actions: apply Helm chart change to observability namespace (values updated), schedule a post-deploy review in Linear ENG-3491.

Notes for oncall: New alerting behavior is enabled in cohorted namespace; if you see unexplained quieting, check /observability/optimizer/status and the cohort-translation table. The runbook includes quick rollback steps (remove cohort export in metrics export config).
Add invocation-level summary rollups and a batching-aware metric optimizer to reduce metric cardinality and lower alert noise. Includes dashboard panels and default alert tuning (quiet-cohort defaults).
Aaron Liu: ensure stable cohort hashing and provide rollback mapping - addressed in aggregator.py and tests
Marisol Vega: do not overwrite raw metrics; export to separate namespace behind opt-in - addressed and documented
Tom Bennett: add synthetic replay harness to validate quiet-window proposals - added under services/batch_metric_optimizer/replay
CI bot: integration tests flaky-network retried once and passed
