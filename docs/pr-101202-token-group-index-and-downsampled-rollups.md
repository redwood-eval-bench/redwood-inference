# Introduce token-group-index and downsampled rollups to reduce cardinality noise in token-level metrics

Motivation: Token-level metrics are extremely useful for attributing cost and latency, but high-cardinality token values and long-tail prefixes generate noisy dashboards and extremely large rollup tables. This PR introduces a token-group index (compact semantic bucketing) plus a downsampled rollup path to reduce storage and alert noise while preserving signal for anomalies and SLO regressions.

Summary of changes: compact token-group index service that maps raw token prefixes to an opaque group id using a hash+semantic-similarity tier; new downsampled-rollup aggregation pipeline that keeps full-resolution samples for heavy hitters and retains statistically representative sampled aggregates for long-tail groups; new instrumentation hooks to annotate spans with token_group_id and sampling_reason; dashboards and alert tuning to use token_group_id by default to reduce cardinality-driven alert fatigue.

Design highlights: (1) Token grouping happens at service edge before ingestion. We use a two-tier policy: deterministic prefix hashing for very short prefixes and a lightweight semantic fingerprint (BERT-lite) for longer prefixes, combined via a stable mapping stored in a compact LRU-backed index. (2) Rollups: heavy-hitter detection keeps exact per-token counters for top-N groups per minute; for remaining groups we emit reservoir-sampled summaries and merge them into downsampled histograms stored as compact protobufs. (3) Backwards compatibility: raw_token remains available behind a flag; all new queries can join on token_group_index when requested. (4) Alerts: token-level SLOs were noisy; updated alerting rules aggregate on token_group_id with adaptive thresholds and a noise-suppression cooldown.

Why this is different from earlier work: prior PRs added rollups and microtrace-synthesis; this PR focuses on upstream grouping and an explicit downsampled aggregation pipeline to reduce cardinality at the ingestion boundary rather than post-hoc aggregation, which reduces ingestion costs and tracing cardinality across both hosted and private deployments.

Checklist:
- [x] Implement token_group_index service and in-process client
- [x] Add span annotation and SDK toggle for token_group_id
- [x] Implement downsampled rollup aggregator and compact storage format
- [x] Update Prometheus/metrics exporter to include token_group labels and sampling_reason
- [x] New Grafana dashboards for group-level cost/latency distribution
- [x] Alert tuning: replace noisy per-token alerts with group-based thresholds
- [x] Unit & integration tests for index, aggregator, and query paths
- [x] Migration plan and feature flag for gradual rollout

Testing and validation: unit tests cover index mapping invariants and sampling correctness (reservoir downsampling tests); integration test validates that for synthetic heavy-hitter traffic we retain exact counters and downstream queries match expected aggregates; load test ran on benchmark-lab with representative traffic for 24h and showed a 4.3x reduction in ingested distinct token labels and a 2.1x reduction in storage for rollup tables at 99th percentile load. CI run included full rt-tests and nightly benchmarks.

Rollout plan: behind feature flag defaulted off. Step 1: enable in hosted staging for 10% of routes and monitor group-level SLOs. Step 2: expand to 50% and enable sampling_reason metrics to validate anomaly detectability. Step 3: flip default for non-sensitive tenants; remaining customers can opt in. Private installer includes opt-in toggle in the helm chart with safe rollback path.

Backward compatibility and privacy: raw token values remain optional; default behavior strips raw_token for retention policies that require minimization. Token grouping intentionally returns opaque group ids (g-xxxxx) to avoid leaking full tokens in dashboards.

Known limitations: edge grouping uses a lightweight semantic fingerprint that is intentionally lossy; some false merges may occur for semantically similar short tokens. We mitigate by keeping heavy-hitter exact counters and providing a manual nomination flow for groups that need to be split.

Followups: (1) Add ML-backed dynamic re-clustering for groups that exhibit drift (ENG-3184), (2) explore an alternative HLL-backed uniqueness sketch for low-memory cardinality detection (ENG-3190).

Related: Linear ENG-3109, ENG-3121. Internal doc: https://redwood.internal/obs/token-group-index (private).
Evan Brooks: Can you quantify the false-merge rate from the semantic fingerprint? We need that to understand alert suppression risk.
Priya Nair: Added a short experiment in the PR description and included synthetic load results — false-merge rate measured ~0.4% on our synthetic corpus for tokens longer than 6 chars. Also added heavy-hitter exact-counters to eliminate impact on top-N items.
Miguel Alvarez: Please ensure the prometheus cardinality doesn’t regress in multi-tenant envs — can we add a per-tenant cap for label explosion?
Priya Nair: Added per-tenant token_group cap in exporter and a guard that drops token_group labels when cap exceeded (configurable), documented in installer values.
Sofia Jensen: Are the group ids stable across restarts? We rely on stability for long-term SLO reporting.
Priya Nair: Yes — mapping is persisted in the compact index with stable hashing and checkpointing. Documented in service README and covered by integration test that simulates restarts.
