# invocation dejitter and token-cost resampler + historical backfill

Motivation: token-level cost and 95th latency alerts have been noisy across high-cardinality routes due to micro-jitter in invocation timings and coarse per-invocation cost attribution. This PR introduces an "invocation dejitter" layer and a token-cost resampler that produces a smoothed, statistically robust per-route cost signal for alerting and billing dashboards. It also adds a one-off backfill job to recompute historical metrics for the last 30 days so SLO alerts and dashboards reflect the new signal.

What this PR does (high level):
- Adds a dejitter pipeline that normalizes inbound invocation timestamps and groups micro-bursts before computing latency percentiles (metric: invocation_dejitter_latency_seconds).
- Introduces a token-cost resampler service that builds a compact sketch of token cost distributions and emits resampled aggregates: token_cost_resampled_total, token_cost_resampler_sketch_count.
- Annotates spans with span attribute "resampled=true" and enriches logs for resampled invocations to aid debugging.
- Adds dashboard panels: Dejittered latency waterfall, Token Cost Resampler heatmap, Resampler sketch inspector.
- Adds alert tuning: new alert rules use the resampled 95th/99th windows to reduce false positives; legacy alerts remain in guarded mode for two weeks.
- Provides a backfill job (scripts/backfill_resampler_job.go) that recomputes resampled metrics for the past 30 days; job admits rate limits and checkpointing to avoid DB overload.

Design notes:
- Dejitter aims to remove sub-100ms timestamp noise from client SDKs and transport reordering. We compute a sliding-window median offset per callsite and adjust the invocation timeline before percentile aggregation.
- Resampler uses a count-sketch + reservoir sampler per route to capture heavy tails while keeping cardinality bounded. Aggregates are emitted at 1m and 1h rollups.
- Tracing: a light-weight span annotation is added at the serving runtime hook. This does not alter span IDs or propagation; it only sets an attribute and emits a small trace event so that sampled traces can be correlated to resampled metrics.

Rollout plan:
1) Deploy services to staging and run synthetic replay (included test harness) for 48h.
2) Enable metrics emission in canary region (us-west-2) and run backfill on a 5% sample to sanity-check results.
3) Enable global rollout and run full 30-day backfill in throttled mode (checkpointing + daily progress reports).
4) After 14 days of canary, flip legacy alerts to disabled and the resampled alerts to primary.

Backfill safety: the backfill writes to a dedicated time-series namespace resampled_* to make rollback trivial. The backfill supports incremental runs and idempotency via job checkpoints.

Checklist: 
- [x] Unit tests for dejitter median estimator
- [x] Integration tests for resampler sketch correctness
- [x] End-to-end replay harness included (tests/e2e/resampler_replay.py)
- [x] Dashboards (dashboards/dejitter_latency.json, dashboards/resampler_heatmap.json)
- [x] Alert rules (alerts/resampled_cost.rules)
- [x] Backfill job with checkpointing (scripts/backfill_resampler_job.go)
- [x] Runbook/incident playbook updated (docs/observability/resampler_playbook.md)

Notes for reviewers: pay special attention to the API between the serving runtime and the dejitter hook (serving/runtime/dejitter.h) and to the correctness of the reservoir-based sketch merging logic. The sketch code path is tested but sensitive to off-by-one sampling biases.

Related: ENG-3456 (observability: reduce alert noise), SUP-7783 (customer report: spurious token-cost alerts).
Ravi Singh: Can we prove the dejitter median estimator is stable under out-of-order arrivals? Please add a small sim in the unit tests.
Author (Evelyn): Added sim (serving/runtime/dejitter_test.cc) covering 0-3s reordering; results deterministic with >1k runs.
Marisol Ortega: The backfill job needs rate limiting; please ensure it respects tsdb write quotas and has a dry-run mode.
Author (Evelyn): Backfill now supports --dry-run and a throttle parameter; checkpointing added to avoid reprocessing.
Tomás Alvarez: Alert thresholds look too tight for some low-volume routes; suggest a grace multiplier and auto-fallback to legacy metric for tiny routes.
Author (Evelyn): Added grace multiplier and a conditional fallback when route sample_count < 10 per minute.
- Reduced alert noise by smoothing invocation timestamp jitter and resampling token cost distributions.
- New metrics: invocation_dejitter_latency_seconds, token_cost_resampled_total, token_cost_resampler_sketch_count.
- New dashboards and alert rules for resampled signals.
- Backfill job recomputes resampled metrics for the last 30 days (run as part of rollout).
If you see sudden drop in token_cost_resampled_total during backfill window, check backfill_resampler_job logs and resume from last checkpoint. Legacy alerts remain enabled in guarded mode until 2026-03-14.
