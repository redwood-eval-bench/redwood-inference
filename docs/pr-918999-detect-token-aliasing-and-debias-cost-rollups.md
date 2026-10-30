# detect-token-aliasing-and-debias-cost-rollups

Context: We discovered repeated cost spikes attributed to single tokens that, upon inspection, were actually groups of visually-different tokens (e.g., punctuation variants, zero-width chars, normalization artifacts) that map to similar model internals and therefore distort token-level cost attribution. This PR introduces a lightweight detector for token aliasing, a debiased rollup path for per-token cost/latency aggregates, and a dashboard + alert tuning to make per-token cost signals more robust.

What this PR does (summary):
- Adds an offline token-alias detector that runs as part of observability ingestion to surface high-alias token families and a per-tenant aliasing score.
- Introduces a debiased_rollup pipeline stage that clusters aliases and re-attributes token-level cost/latency to canonical alias groups when computing rollups used by dashboards and SLO monitors.
- Emits three new metrics: obs.token_alias_score (gauge), obs.alias_group_size (gauge), obs.debiased_token_cost{alias_group} (counter-like).
- Dashboard: new panel under Observability -> Token Economics: alias heatmap, alias-group cost breakdown, and SLO confidence bands.
- Alert tuning: updates the per-token cost alert rule to use alias-aware aggregation and a new quieting operator that reduces repeat alerts caused by alias churn.
- Tests: unit tests for detector heuristics and integration tests for the debiased rollup using synthetic replaybench traces.
- Docs: developer-facing doc page describing aliasing, mitigation rationale, and how to opt-out per-tenant.

Checklist: 
- [x] detector implementation and unit tests
- [x] rollup pipeline integration and end-to-end integration tests
- [x] dashboard JSON and dashboard docs
- [x] alert rule & playbook update
- [x] basic performance safety checks (adds ~0.2ms median per-invocation ingest on dev cluster; opmit later if needed)

Motivation & trade-offs:
Token aliasing is relatively rare but causes high-precision observability signals to misattribute cost and generate noisy alerts. The chosen approach keeps detection offline/ingest-side and only changes downstream rollups used for monitoring and dashboards. We intentionally do not change raw per-invocation traces or billing primitives. This preserves billing fidelity and keeps the change scoped to observability. There is an opt-out flag for tenants who prefer raw attribution.

Related work: ENG-4312 (investigate token-level false positives), ENG-4320 (dashboard SLO confidence work).

How to validate:
- Run replaybench with token-alias fixture (tests/integration/alias_replaybench) and verify debiased_token_cost aligns with expected cluster attribution.
- Load Observability -> Token Economics and confirm alias heatmap appears and group sizes are sane.
- Trigger synthetic alias churn (tools/obs/sim_alias_churn) and observe alert rate reduction compared to baseline.

Rollout plan:
- Canary deploy the ingestion change to dev and staging for 48h.
- Enable debiased rollups for 5 internal tenants for 7 days.
- If metrics are stable, enable platform-wide in monitoring (no billing change).

Operational notes:
- Alias detector config lives in obs-ingest/config/alias_config.yaml (defaults tuned conservatively).
- There's a tenant opt-out via tenant flags: observations.aliasing_enabled=false.

This change is focused, reversible, and includes dashboards/playbooks to help SREs understand when aliasing is the cause of cost shifts.
Add detection and mitigation for token aliasing in cost rollups; new metrics and dashboard; alert sensitivity tuning that reduces false positives for cost spikes attributed to aliasing.
Diego Ramos: The detector heuristics look reasonable but can we add a short-circuit for extremely common tokens (stoplist) to avoid noisy clusters?
Maya Chen (author): Added a stoplist and a note in alias_config.yaml; unit tests updated to cover stoplist behavior.
Katherine Liu: Integration tests are great. Concern: ingest latency shows a 0.18ms median increase on staging - can we add a configurable sampling window?
Maya Chen: Added sampling window config and a benchmark note in docs. We'll monitor during canary.
Omar Haddad: Alert rule looks good. Suggest including a brief playbook snippet to explain expected alias-group sizes for oncall triage.
Maya Chen: Added playbook section and an example runbook in docs/observability/token_aliasing.md#playbook.
