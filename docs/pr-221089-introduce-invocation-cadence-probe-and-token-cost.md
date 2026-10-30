# Introduce invocation cadence probe and token-cost ribbons

Adds an invocation cadence probe metric, token-cost ribbon panels in Console, and alert tuning for short-burst invocation churn. No public API breaking changes.
Context: We frequently see short, high-frequency invocations from chat UIs and background agents that create bursty token-cost spikes and noisy alerts. Root cause analysis is slow because existing metrics aggregate over invocations and hide cadence (inter-invocation intervals) and per-token cost structure.

Goal: Add a lightweight invocation-cadence probe that emits a small set of metrics and span attributes to help: 1) surface burstiness patterns (invocation-per-second ribbons), 2) attribute token-cost ribbons across bursts, and 3) reduce alert noise by tuning short-burst SLOs.

What this PR does: 
- Telemetry agent: new cadence probe that samples invocation timestamps (probabilistic, low-overhead) and exposes three metrics: redwood_invocation_cadence_histogram (exponential buckets for inter-arrival ms), redwood_invocation_burst_count (per-window burst counter), and redwood_token_cost_ribbon (compact rolling histogram indexed by invocation cadence buckets).
- Tracing: adds span attributes invocation.cadence_bucket and invocation.burst_id for sampled spans. These are optional attributes added only when cadence probe is active to keep trace size small.
- Console dashboards: new Token Cost Ribbons and Cadence Overview panels (token_ribbons.json) that render per-cadence ribbons and offer a toggle to overlay service-level SLOs. Panels are small, performant Vega-lite based visualizations with pre-aggregated rollup queries.
- Alert tuning: adds a new alert rule (alerts/short_burst_slo.rules) that suppresses token-cost SLO alerts when the burst pattern matches benign cadence signatures (configurable decay window). Also introduces a per-route short-burst exemption list configurable from Console.
- Tests and docs: unit tests for cadence bucketing, an e2e synthetic replay test to validate ribbons against golden data, and doc snippets in docs/observability/cadence-probe.md.

Checklist:
- [x] Add metrics endpoint and probe implementation
- [x] Expose span attributes guarded by sampling
- [x] Console dashboard panels + small UX toggle
- [x] Alert rule and configuration knobs
- [x] Unit and integration tests
- [x] Inline docs and migration notes

Commits (summary):
- feat(observability): add invocation cadence probe + exporter (author: Maya Patel)
- feat(console): token-cost ribbons dashboard + queries (author: Maya Patel)
- perf(telemetry): reduce cadence probe overhead and add probabilistic sampling (author: Maya Patel)
- test: add replaybench and unit tests for cadence bucketing (author: Maya Patel)

Changed files (high level):
- redwood/telemetry/cadence_probe.go - new probe, histogram & ribbon exporter (low-level logic)
- redwood/tracing/span_enricher.go - add invocation.cadence_bucket attribute (guarded by sampling)
- redwood/console/dashboards/token_ribbons.json - new dashboard panels and example queries
- redwood/alerts/short_burst_slo.rules - new alert rule with suppression logic
- redwood/tests/replaybench/cadence_golden_test.py - e2e replay test that validates ribbons
- docs/observability/cadence-probe.md - documentation and migration suggestions

Notable implementation details:
- Cadence probe maintains a tiny ring buffer of last N invocation timestamps per route shard and computes inter-arrival ms; only emits aggregated counts to avoid high cardinality. Sampling rate defaults to 1% across routes; rate is configurable per-tenant and per-route.
- Token-cost ribbon metric is a compact rolling histogram keyed by cadence bucket (low-cardinality: 8 buckets) and emits periodic rollups to the metrics pipeline. This design keeps storage costs bounded while letting Console render ribbons with a single aggregation query.
- Span attributes are only attached for sampled traces (sampling decisions reused) to avoid trace bloat. Attribute values are coarse (e.g., cadence_bucket=fast/medium/slow) to minimize cardinality.

Review conversation (selected):
- Lucas Chen: "Overall this looks great — can you justify the default 1% sample rate and add a test that proves the ribbon shape is stable under sampling?"
- Maya Patel: "Added replaybench test that replays synthetic bursts with downsampling and compares ribbon quantiles to the full sample; math and thresholds are in docs. Default 1% chosen based on perf testing (see perf note in cadence_probe.go)."
- Ravi Desai: "Alert suppression looks promising; please ensure the suppression window is keyed by tenant and route and document how to opt out."
- Maya Patel: "Suppression keys updated; added Console opt-out toggle and documented behavior."
- Elena Novak: "CI flake on metrics exporter; please ensure exporter retry/backoff doesn't amplify bursts."
- Maya Patel: "Fixed exporter backoff in second commit and added a unit test to assert exponential backoff bounds."

CI notes: initial CI run reported a flaky metrics integration test (timeout on testrun), subsequent rerun passed. All checks green before merge: unit, integration, lint, e2e replaybench.

Rollout & migration notes:
- Probe disabled by default for tenants with < 1k invocations/day to avoid unnecessary sampling.
- Existing alert rules remain unchanged; short-burst suppression is additive and must be enabled per-tenant or by account-level config.
- No schema changes to emitted metrics; token_cost_ribbon uses existing compact-histogram encoding we already support.

Backward compatibility: No breaking changes to public API. Tracing attributes are additive and optional.

Follow-ups (planned):
- ENG-5002: Add a console annotation that links a ribbon segment to a sampled trace collection for deeper RCA.
- ENG-5011: Automated knob that suggests per-route sampling rate based on historical cadence variability.

Merge outcome: Squash merged into main on 2025-06-20 after approvals from Lucas Chen and Ravi Desai and a final sign-off from observability oncall.
