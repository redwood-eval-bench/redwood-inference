# Introduce token micro-profiling and alert churn attenuator

This PR introduces per-token micro-profiling (microprofiles) for invocation lifecycles and an "alert churn attenuator" that reduces noisy SLO alerts caused by token-level variance spikes. Motivation: incidents in Jan-Feb showed many high-volume tenants triggered transient token-latency alerts due to short-lived prompt shapes (warm/hot prefixes). We need better granularity to attribute latency to token phases and to suppress alert churn without hiding real regressions.

What this PR does (high level):
- Adds a lightweight in-memory microprofile collector that samples token phase latencies (prefix, generation, kv-lookup, postprocess) at 1-in-500 invocations with reservoir sampling by tenant and route.
- Emits two new metrics: telemetry.token_microprofile.hist (histogram of per-phase token latencies, labels: tenant, route, phase) and telemetry.token_microprofile.signature (compact fingerprint of prompt-shape) for rollups.
- Introduces a new alerting circuit (alert-churn-attenuator) that: computes short-window alert rate per-tenant, suppresses duplicate alerts if correlated with known prompt-signatures, and escalates only when median microprofile latency increases persistently across a decaying window.
- Adds dashboard panels: Token Microprofile heatmap, Signature collision explorer, and Alert Churn monitor.
- Hooks the pipeline into existing trace enrichment so sampled traces include microprofile spans and signature annotations.

Files touched (summary):
- runtime/src/telemetry/token_microprofile.rs: core collector, sampling policies, labels. (+220 +)
- runtime/src/telemetry/metrics_exporter.rs: histogram bucket mapping and exporter hooks. (+48 +)
- console/dashboards/token-microprofile.json: new Grafana panels and panel descriptions. (+40 +)
- infra/alerting/attenuator_rules.yaml: attenuation rules and suppressors. (+18 +)
- runtime/src/tracing/span_enricher.rs: attach microprofile context to sampled spans. (+52 +)
- sdk/python/redwood/telemetry.py: SDK helper to surface signature to customers. (+24 +)
- tests/integration/test_microprofile_endtoend.rs: end-to-end sim of routing, sampling, and alert suppression. (+10 +)

Commits:
- c1a7f2a: add sampler + reservoir implementation and unit tests
- f8d4b3e: wire histogram exporter and add buckets
- 9b2e1d1: add signature computation and SDK helper
- 2c7d4a5: dashboards and alert attenuation rules
- 5e1b9c0: end-to-end integration tests and CI adjustments

Review / discussion highlights (single string for history):
- Marcus: asked for guarantees around sampling collision risk and overhead; I added bounded-reservoir per-tenant and CPU budget guard, and included microbenchmark numbers in comments of token_microprofile.rs.
- Sofia: requested that attenuator not suppress alerts for tenants with SLA=enterprise; implemented tenant-tier override and added label 'tier' to metrics.
- Rafael: noted potential for metric cardinality explosion; we clamp signature cardinality by hashing and exponential decay; added docs to console panels explaining aggregation windows.
- Author replies: responded with additional unit tests, included cost impact estimate (~0.3% CPU at 1/500 sampling) and rollback plan.

Testing & CI:
- Unit tests: token_microprofile unit suite passing locally and on CI.
- Integration: test_microprofile_endtoend simulates 10k invocations, sampling stable across repeats.
- Load: microbench shows <1% p95 CPU increase at qps=2000.
- CI: initial run failed lint on console JSON formatting; fixed and re-ran — final CI status: pass.

Rollout plan:
1) Merge to main behind feature flag TOKEN_MICROPROFILE (default off for hosted tenants).
2) Enable for internal staging and run synthetic SLO replay for 48h.
3) Enable for opt-in customers and monitor Alert Churn monitor for 7 days.
4) Ramp to 5% tenants, 25% tenants, then global if stable.

Fallback / Rollback:
- Feature flag to disable collector; rollback path is to flip off TOKEN_MICROPROFILE and remove attenuator from alerting rules.
- Metrics and dashboards are additive; removal reverts new panels but does not alter existing metrics.

Release notes:
- Added token microprofiling (sampled) and Alert Churn Attenuator to reduce noisy token-level alerts. Opt-in via feature flag. See console docs for dashboard walkthrough.

Related Linear/Jira: ENG-11402 (alerting hygiene), ENG-11670 (token observability)
Marcus Liu: Can we bound sampling memory and explain collision risk? Added reservoir and comments; microbench attached.
Sofia Martín: Ensure enterprise tenants aren't suppressed. Added tenant-tier override and 'tier' metric label.
Rafael Ortega: Cardinality concern—how do we avoid explosion? Clamping and decay strategy added; doc note on panels.
Anika Rao (author): Added additional tests and CPU cost estimate; PTAL — ready for final review.
