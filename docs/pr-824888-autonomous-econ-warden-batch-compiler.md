# Autonomous econ warden: batch template compiler and quant slimming pass

Motivation: We need automated, tenant-safe defaults for batching/caching and conservative quantization trials so that small customers get cost wins without SLO regressions. This PR introduces the Autonomous Econ Warden: a service that compiles workload-aware batch templates, runs a staged quant-slimming pass against sampled traffic in shadow mode, and ships conservative defaults into the tenant defaults registry. Key goals: 1) produce compact batch/caching templates tuned by tenant profile; 2) try compact quant profiles automatically with safety rollbacks; 3) provide a CLI/operator surface for dry-runs and repro; 4) emit unit-econ artifacts for auditing and downstream simulation. Changes are intentionally opt-in via a policy flag and the warden runs in shadow by default.

What this PR does (high level): - adds pkg/econ/warden: core orchestration, template compiler, safety gates - adds cmd/econctl: CLI for simulate/dryrun/apply - adds configs/econ-templates: templating DSL and a small corpus of starter templates - introduces quant/compatibility probe: a staged, checksum-based trial runner that validates loss/latency budgets - adds unit-econ simulator that produces expected unit-cost deltas for each candidate template/profile - hooks: telemetry & audit events for each decision and replay traces for offline verification - tests: BDD e2e simulated traffic harness and unit coverage for the compiler

Checklist: - [x] core compiler and template DSL implemented - [x] shadow quant-slimming pipeline integrated with rollout safety checks - [x] telemetry events and dashboards (ops) wired - [x] CLI for dry-run / apply / rollback - [x] e2e tests and integration job added - [x] docs/ops/cost-automation.md added

Operational notes: - Default behavior is shadow-only. To enable auto-apply, set policy.auto_apply=true per-tenant. - Safety gates: max 1% delta in top-1 accuracy proxy and 10ms p99 latency regression on sampled workload; else template is rejected. Rollback uses a 24h post-apply observation window and can be forced by ops via econctl revert. - Telemetry: events/econ_warden/decision_{try,apply,reject,rollback} emitted; unit_econ reports include expected_cost_delta_usd and variance_estimate.
Khaled Hassan: Nice design. Two asks: (1) can we constrain template compiler to use tenant SLO metadata so we don't produce batch shapes that will violate p99 targets? (2) Add a kill-switch env var for immediate disable.
Marina Vega: Addressed. Compiler now pulls tenant_slo.target_p99 and respects a max_batch_latency budget; added ENV ECON_WARDEN_DISABLED for quick kill.
Jenna Park: Unit-econ numbers look promising. Can you add a small randomized A/B test harness for the probe so we can sanity-check on canary tenants?
Marina Vega: Added canary selection logic (configs/econ-templates/canary.yaml) and test harness; included sampling seed for reproducibility.
Samir Rao: CI flaked on initial integration run due to test timeout; once extended the timeout everything passed. LGTM after resolving the lint nits.
Khaled Hassan: Approved
Jenna Park: Approved
Samir Rao: Approved with one comment about telemetry thresholds which is addressed in a follow-up ticket ENG-2610.
