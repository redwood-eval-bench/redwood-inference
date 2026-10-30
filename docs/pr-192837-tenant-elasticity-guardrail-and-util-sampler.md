# Introduce tenancy elasticity guardrail and utilization sampler

Introduces an elasticity guardrail controller that enforces tenant elasticity caps and a background utilization sampler for percentile telemetry. Improves stability under tenant surges and provides per-tenant utilization histograms for SLO tuning.
Motivation: tenants with bursty traffic occasionally trigger aggressive autoscaling which leads to noisy neighbor effects and uneven performance across the fleet. This PR adds an "elasticity guardrail" that limits short-term elasticity across dedicated pools while preserving throughput guarantees for steady-state load. It also introduces a lightweight utilization sampler that emits per-tenant p50/p90/p99 utilization estimates to drive better autoscaler decisions.

What this changes: the new controller maintains a per-tenant ephemeral credit bank (elasticity credits) that accrues slowly under steady usage and decays over time. Rapid ramp-ups consume credits from a local surge buffer; when credits deplete the admission layer enforces a soft throttle that prefers degraded-but-safe execution paths (lower-concurrency slots, model-variant fallbacks, or queued tokens) before requesting cluster-wide provisioning. The utilization sampler collects rolling sampled histograms and reports percentiles to the control plane at 10s granularity.

Design details: credit accrual rate is configurable via the pool config (fields: elasticity.credit_rate, elasticity.max_bank, elasticity.decay_tau). Consumption is measured in concurrency-seconds to align with tenant throughput billing. Admission policy is a prioritized set of fallbacks: 1) reduce concurrency reservation, 2) use quantized model variant, 3) queue request with capped wait. The guardrail is independent of reserved capacity so steady commitments are unaffected.

Testing & validation: unit tests for credit arithmetic, integration tests that exercise admission fallbacks, and a synthetic loadbench that runs 24-hour replay traces from a large tenant. Locally reproduced the production incident (SUP-1128) and verified that the guardrail reduced noisy neighbor evictions by ~78% in the trace. Microbenchmarks: median latency unchanged, p99 latency decreased by ~3.2% for the fleet under mixed workloads; service-level cost impact was a ~7% decrease in emergency provisioning events.

Checklist:
- [x] Controller implementation and reconciliation loop
- [x] Admission layer hooks and prioritized fallback paths
- [x] Telemetry sampler and export mapping
- [x] Config knobs and K8s CRD fields documented
- [x] Unit + integration tests
- [x] Loadbench simulation and perf summary attached

Rollback plan: the guardrail is feature-flagged and controlled by a cluster feature toggle (feature.elasticity_guardrail). Rolling the flag to OFF disables both admission enforcement and credit accounting. CRD additions are additive and safe to remove if needed.

Related work: follows on from ENG-5432 which defined the admission fallback matrix. Complements existing reservation and warm-pool controllers but is intentionally conservative (no automatic capacity pre-provisioning).

Notes for reviewers: focus on correctness of credit decay math, admission ordering semantics, and tagging of telemetry metrics to avoid high-cardinality explosions. The sampler intentionally bins tenants into cohort buckets for histograms to limit metric cardinality.
2026-02-15T09:12Z - Ethan Park: Nice design. Two questions: how do we prevent metric cardinality explosion from the sampler? Also can we express credit units in a less confusing way than concurrency-seconds?
2026-02-15T11:03Z - Priya Menon: Reply: sampler buckets tenants into configurable cohort groups (small/medium/large) and we emit percentile aggregates per-cohort+pool only; full per-tenant histograms are opt-in for debugging. Credit units are now documented as 'concurrency-seconds' with examples; open to alternate naming if you have one.
2026-02-16T14:27Z - Lina Zhou: In the admission path, if we choose queued tokens as fallback we must attach a TTL to avoid head-of-line blocking. Please add a queue TTL check.
2026-02-16T15:10Z - Priya Menon: Added queue TTL enforcement in admission path and a unit test that verifies we drop stale queued requests.
2026-02-17T08:44Z - Marco Alvarez: Observability looks good; would like a dashboard tile for elasticity credit health (available vs consumed) per-pool.
2026-02-17T12:02Z - Priya Menon: Added metric guardrail_credit_{available,consumed} and updated dashboards in docs/ops to include suggested Grafana panels.
2026-02-19T20:01Z - Ethan Park: Approve after addressing minor comment about magic constants in decay math.
(Resolved)
