# Introduce hierarchical prefill autoscaler and quantization telemetry bridge

Motivation: Tenant workloads show multi-modal request patterns where a single global prefill/batching policy either overspends or underperforms for different cohorts. This PR implements a hierarchical prefill autoscaler that composes tenant-level and global-level hints to produce safer defaults, plus a telemetry bridge that captures quantization-era metrics (post-quant latency/error deltas) so the optimizer can recommend conservative quant tiers.

What this PR does: Adds a new batch hierarchy controller in the serving runtime which: (1) evaluates tenant prefill profiles, recent cold-start rates, and latency SLO violations; (2) computes an autoscale suggestion that can be applied as a non-disruptive default or promoted to tenant overrides; (3) attaches conservative quantization recommendations to suggestions based on observed latency/error deltas captured by the telemetry bridge.

Key characteristics:
- Tenant-first but fallback-friendly: tenant hints are preferred; when missing we synthesize a safe global default.
- Conservative quant tiers: telemetry bridge measures inferred token-level latency and token-loss proxies after quant changes and uses a small exponential backoff to avoid flapping.
- Dry-run and safety: introduced a rwctl prefill-sim dry-run that shows expected unit-econ impact and SLO risk score before applying policies.
- Minimal runtime overhead: hierarchy eval runs on the batch controller worker thread with a 1s cadence and uses cached stats to avoid hot paths.

Files touched (high level): runtime/batcher/hierarchy_controller.go, runtime/batcher/hierarchy_policy_test.go, telemetry/quant_bridge/collector.go, telemetry/quant_bridge/proto/quant_telemetry.proto, cmd/rwctl/prefill_sim.go, examples/prefill-samples/tenant_profiles.yaml, docs/cost-automation/hierarchical-prefill.md, CHANGELOG.md.

Testing and validation: Unit tests for hierarchy scoring and quant-safety logic (coverage + edge cases for empty tenant history); integration test that runs a tiny local workload and asserts decision output and no regression in tail latency; manual smoke runs on staging with two tenants to validate dry-run numbers against simulated billing. CI: all checks passed (go test, linter, telemetry proto compile).

Rollout plan: merged behind a feature flag (feature.prefill_hierarchy) with default off for existing tenants. Stage rollout: enable in staging -> opt-in beta tenants -> gradual global rollout with metric gates (increase in 95th latency or token loss > 0.5% triggers automatic rollback).

Related: ENG-4872 (prefill autoscale design), ENG-5900 (quant telemetry schema), small follow-up planned to surface suggested defaults in the Console (separate PR).
Maya Li: Nicely scoped. Two asks before approving: (1) add a short circuit if tenant history is < 30 requests to avoid noisy decisions; (2) ensure the telemetry proto versioning has migration notes. Can you address?
Sofia Nguyen (author): Added the short-circuit and a unit test for low-history tenants, proto now includes a version enum and migration notes in docs. PTAL.
Daniel Park: One nit on variable naming in hierarchy scoring, otherwise lgtm. Also can we add a quick perf benchmark for the 1s cadence?
Sofia Nguyen (author): Renamed variables and added a micro-benchmark in runtime/batcher/bench_hierarchy_test.go showing ~120ns per evaluation on avg. CI green.
Omar Patel: Verified dry-run numbers on staging with two tenants. Suggested adding a safety gate when token-loss delta > 1% (we keep 0.5% for now but tracking the gate).
Sofia Nguyen (author): Noted in rollout plan and will follow up with the Console PR to surface the gate to SREs.
Add hierarchical tenant-aware prefill autoscaler and quantization telemetry bridge; introduce rwctl prefill-sim dry-run to estimate unit-econ impact. Feature gated behind feature.prefill_hierarchy. See docs/cost-automation/hierarchical-prefill.md for usage.
