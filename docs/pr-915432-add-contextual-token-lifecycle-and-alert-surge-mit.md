# add-contextual-token-lifecycle-metrics-and-alert-surge-mitigator

Motivation: our alerting surface has many noisy alerts triggered by short-lived token bursts (e.g., retries, prompt reshapes, prefetch storms). These bursts create correlated traces that are operationally uninteresting yet cause pager fatigue. This PR introduces (1) a contextual token lifecycle metric set (ingest -> prefix-lookup -> kv-load -> decoder-start -> generation-end) and (2) an alert surge mitigator that uses a temporal correlation matrix and cohorting heuristics to suppress transient multi-route alarm storms while preserving true SLO violations.

What changed: Added a lightweight lifecycle event emitter inside the serving runtime and a metrics schema v2 that encodes phase and cohort tags. We enrich traces with a new span.tag `token.phase` and emit phase durations as histogram+sum for downstream aggregation. The alert mitigator runs as part of the observability pipeline and computes a rolling temporal correlation matrix per-tenant (configurable window) to detect high-coherence token bursts. When a surge is detected, the mitigator applies a short automatic snooze (with exponential backoff) to non-rooted alerts and flags an incident hint for on-call.

Key benefits: - Reduced alert noise for token-burst patterns observed in ENG-2104. - Better attribution: per-phase breakdowns allow us to see if cost/latency is dominated by KV loads vs generation. - Faster RCA: trace enrichment links lifecycle events to cache hits/misses and kernel queues.

Rollout plan: feature flagged behind `telemetry.token_lifecycle_v2` and `observability.alert_surge_mitigator`. Default rollout is 0% in hosted, 50% in internal staging, 100% in production after one week of stable metrics and no false-suppression incidents. Migration notes: metrics schema v2 is additive; downstream dashboards are updated and a single migration job will backfill phase histograms for the last 48h in case of partial data.

Testing: unit tests for lifecycle emitter and mitigator heuristics, e2e run against replaybench with synthetic burst scenarios (included). CI surfaced a lint failure on first run which was fixed in the second commit.

Related: ENG-2104 (noise reduction epic), SUP-4321 (customer report of paging spike).
Daniel Kim: Can you add a short unit test that asserts the mitigator does not snooze alerts for sustained SLO violations? Reason: we mustn't mask real incidents (requested in ENG-2199).
Priya Venkataraman: Added a regression test in tests/e2e/token_lifecycle_replay_test.go that simulates sustained vs transient bursts. See commit 9f8e7d6.
Marisol Ortega: The dashboard looks good; please add a short paragraph in docs about interpreting phase ratios (kv vs decoder).
Priya Venkataraman: Documentation updated in docs/observability/token-lifecycle.md. Also included an example query for phase ratio drift detection.
Ethan Zhang: CI flake on pipeline test — observed non-determinism in timing thresholds. Suggest making window configurable in tests.
Priya Venkataraman: Made the window configurable and added deterministic clocks to tests; CI passes now.
Introduce token lifecycle (ingest/prefix/kv/generate) metrics and an alert surge mitigator to reduce transient alert storms. Added dashboards and migration guidance. Feature gated.
