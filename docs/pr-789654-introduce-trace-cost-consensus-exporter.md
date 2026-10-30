# Introduce trace-based cost-consensus exporter and deferred log bundler

Summary: Adds a trace-level exporter that merges signals (spans, logs, prefetched kv-cache markers) to produce a consensus per-invocation cost estimate and a confidence metric. Also implements a deferred log bundler to attach delayed logs to traces during offline enrichment. Motivation: long-tail cost variance is often caused by mismatched signals (e.g., client-side chunking, backend retries, kernel preemption). Single-source cost attribution can be noisy; this change computes a consensus across available signals to reduce churn in cost-driven alerts and provide a confidence band for downstream dashboards and SLOs.

Design: The exporter ingests sampled traces and runs a lightweight consensus algorithm that: 1) normalizes per-signal token count and HW cost tags, 2) computes a weighted median cost per invocation where weights are derived from signal freshness, kernel-level occupancy and kv-cache hit probability, and 3) emits a cost_consensus metric with tags {tenant, route, model_variant, confidence_bucket}. The algorithm uses deterministic tie-breaking and a compact delta format to make payloads small. Deferred log bundler pairs late-arriving logs with trace anchors using a reservoir of recent invocation anchors and a small LRU index to avoid unbounded memory.

Files/Artifacts: - a new exporter package: observability/exporters/cost_consensus.go (main algorithm + exporter shim) - deferred log bundler: observability/enrichment/deferred_bundler.go - dashboard: dashboards/consensus-cost-explorer.json (panel + confidence band sparkline) - SDK hook: sdk/python/redwood/observability.py changes to expose cost_consensus events to customers via telemetry hooks - tests: tests/integration/test_cost_consensus.py and unit tests for tie-breaker and bundler - docs: docs/observability/cost_consensus.md

Testing and validation: Unit tests cover normalization, weight schedules, and tie-breaking. Integration tests run synthetic replaybench traces (coldstart, prefetch, retry scenarios) and compare consensus output to oracle (replayed ground truth). We ran a 48h dark launch against 5% of production traffic in a staging namespace; consensus reduced per-tenant cost alert churn by ~34% and tightened the 95th percentile cost variance by ~18% in observed traces.

Rollout plan: Feature gated behind OBS_COST_CONSENSUS; rollout will start with read-only metrics (emit to internal metrics store without affecting billing) and then be turned on for alerting consumers only after one week of monitoring. Migration notes are in docs/observability/cost_consensus.md.

Related: ENG-4123 tracked the initial design and evaluation; SUP-9123 documents a customer case where inconsistent logs caused billing mismatch tickets.

Notes on backward compatibility: Exporter emits an opt-in metric namespace (redwood.cost_consensus). No changes to billing pipeline are included in this PR.

Luca: This looks promising; please call out the memory bounds for the LRU reservoir and a safety knob. / Maya: Added reservoir_size and max_attachment_age feature flags; documented default bounds (16k anchors, 2h max).

Priya: The weighted median approach is clever. How are weights computed at runtime? Concern about weighting biasing towards kernel-reported costs when kernel data is stale. / Maya: Weight schedule now factors in 'signal_freshness' and 'kv_hit_rate' with a decay function; added unit test 'stale_signal_bias_test' proving worst-case bias < 5%.

Owen: CI flaked once on replaybench; please re-run and add a retry to the flaky test. / Maya: Increased timeout and guarded the integration test with a retry wrapper; CI stabilized.

Priya (approval): Approve after the doc clarifies how customers can opt-out. / Maya: Added opt-out examples and feature flag usage to docs.

Merge notes: Squash-merged after approvals. Post-merge: enabled read-only metric emission in staging and created a dashboard tile to monitor confidence_bucket distribution.
Add a trace-based cost consensus exporter that emits redwood.cost_consensus metrics and a deferred log bundler for late-arriving logs. Feature gated and opt-in; reduces alert churn by consolidating noisy signals. See docs/observability/cost_consensus.md for rollout and opt-out instructions.
