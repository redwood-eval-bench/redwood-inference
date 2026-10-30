# adaptive-defaults-fabricator-mini-batcher-quant-polisher

Motivation: Many tenants run suboptimal inference economics because defaults for batching, KV-prefill, and quant profiles are static or hand-curated. This PR introduces a lightweight 'defaults fabricator' service that synthesizes tenant-scoped default configs (mini-batcher shapes, cache fallback thresholds, and quantization profile hints) using observed workload signals and an internal heuristic model. The goal is to reduce unit-costs while preserving latency SLOs and to make safe, explainable default changes that operators can opt into.

Summary of changes: implemented a new econ/fabricator service + CLI; new mini-batcher controller that exposes shape suggestions; quant-polisher that post-processes quant profiles to avoid brittle fallbacks; telemetry hooks and evals to measure unit economics impact; feature flag gating and canary rollout plan.

Design notes: the fabricator is intentionally conservative: it synthesizes candidate defaults, scores them with a unit-econ estimator, simulates 1h of sampled traffic (local replay) and outputs suggestions only when estimated 5-10% unit-cost improvements with <3% latency tail uplift. The quant-polisher enforces compatibility rules (min FP16 fallback points, kv-cache alignment) to prevent regressions on long-sequence workloads. Suggestions are persisted as tenant-default artifacts (YAML) and surfaced in the Console as 'Suggested Defaults'.

Checklist: - [x] Implement core synthesizer and scorer - [x] Add CLI for dry-run and export - [x] Telemetry events for suggested/accepted/rejected defaults - [x] Unit and integration tests for the synthetic replay harness - [x] Safety gates: rollback TTL and canary percent gating - [x] Docs and release notes draft

Files touched (high level): repo/src/econ/fabricator.py, repo/src/econ/polisher/quant_polisher.py, repo/src/batcher/mini_batch_controller.py, repo/cli/econctl.py, repo/integrations/telemetry/events.yaml, repo/tests/econ/test_fabricator_replay.py, repo/docs/ops/prefill-defaults.md, deployment/k8s/featureflags.yaml, examples/tenant-defaults/sample_suggestions.yaml

Testing and CI: the first CI run had a lint failure in a CLI arg parser which was fixed in a follow-up commit. Integration tests run a synthetic 1k request replay across typical tenant shapes; results are captured as artifacts and uploaded to the build. Final CI run: build ✅, unit tests ✅, integration replay ✅, e2e smoke ✅.

Rollback and compatibility: all suggestions are gated by a runtime feature flag (econ.defaults.synth.enabled) and a tenant can accept/reject via Console or econctl. The fabricator never mutates active runtime defaults without explicit tenant acceptance.

Telemetry and observability: added econ.fabricator.suggestion_emitted, econ.fabricator.suggestion_scored, econ.fabricator.suggestion_applied, with tags for tenant_id, score_delta, simulated_latency_p95_diff. Dashboards and alerts added to Console for suggestion success rate and latency regressions.

Performance: the default offline replay scaffolding is CPU-bound and runs in <90s for 10k sample requests in our test harness; scoring is single-threaded but sharded per-tenant in production. The mini-batcher controller exposes suggested batch shapes which the serving runtime can opt-in to for a canary pool.

Security and privacy: fabricator runs on tenant-metric aggregates only; no raw PII or request payloads are stored. Suggestions are encrypted at rest and audit-logged.

Rollout plan: 1) internal dogfood on 10 tenants (rollout flag) 2) expand to 100 tenants with monitoring 3) make opt-in prompt in Console 4) after 6 weeks, allow auto-apply for tenants with confirmed stable improvements.

Notes: This PR is intentionally conservative with safety-gates; future work will introduce an ML-based suggestion ranker, but that will be landed behind a separate feature flag and policy review.
Adds an adaptive defaults fabricator that synthesizes tenant-level batching and cache fallbacks plus a lightweight quantization profile polisher. No customer-visible API changes. Default rollout behind flag.
Liam O'Connor: Great foundational work — please add more unit tests around edge cases for long KV cache lifetimes.
Priya Nair: Added additional tests for KV spillover and extended coverage in test_fabricator_replay.py (commit aef9c21).
Mei Zhang: Requesting a follow-up to add a canary inspector that validates suggestion before apply; prefer to keep this out of the initial roll. -> ACK by author; tracked in ENG-1933.
Carlos Rivera: CI failed on first run due to a lint error in econctl; resolved. Also recommend limiting replay sample size default to 10k (not 100k) to bound runtime costs.
Mei Zhang: Approved pending small doc clarifications.
Liam O'Connor: Approved.
