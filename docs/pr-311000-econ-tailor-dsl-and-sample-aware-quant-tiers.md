# Introduce econ-tailor DSL and sample-aware quant tiers with simulator

Problem: customers running highly bursty request patterns (microbursts) were seeing sporadic unit-cost spikes and suboptimal quantization choices because the current defaults are static and tuned for average throughput. This PR introduces a small, embeddable DSL (econ-tailor) for describing sample-aware cost profiles and a CLI-driven dry-run simulator that produces recommended quantization thresholds and batching defaults per-tenant workload samples.

What this PR does (high level):
- Adds pkg/econ/dsl: a concise DSL to express sample-aware rules (histogram triggers, percentile bucketing, warmup windows). The DSL compiles to an intermediate policy that the optimizer can consume.
- Adds tools/cli/econ-tailor: a new CLI that accepts workload sketches (sampled latencies / sequence length distributions / token histograms) and emits recommended: batch shapes, kv-cache retention window, and quantization thresholds (per-model family).
- Adds a simulator (tools/sim/econ-dryrun) that can evaluate suggested defaults against synthetic microburst traces and output unit-econ delta reports. This is used by our CI smoke test to prevent regressions.
- Integrates telemetry hooks (pkg/telemetry) to tag recommended profiles with stable fingerprints and risk scores for rollout gating.
- Adds an opt-in Linear ticket annotation flow: econ-tailor outputs a suggested change JSON that can be attached to Linear / a changelist for easy review by SRE/CostOps.

Why this approach:
- Rather than hardcoding heuristics in the runtime, the DSL lets operators codify how workload samples should map to operational levers (batching, caching, quant thresholds). The dry-run simulator provides confidence by predicting token-unit cost and latency percentiles under microburst mixes.

Checklist: 
- [x] DSL grammar and compiler
- [x] CLI to accept sketches and emit profiles
- [x] Simulator + unit/e2e tests
- [x] Telemetry tagging + fingerprinting
- [x] Docs and quickstart
- [x] Smoke CI job to run a small dry-run on merges

Related items: Linear: ENG-4021 (tenant microburst analysis), ENG-4077 (quant thresholds roadmap). Related PRs: none directly; this is a new entrypoint for cost automation.

Notes for reviewers: focus on the DSL safety gates (no automatic rollout by default), the simulator assumptions (see simulation/README in the diff), and telemetry cardinality (we intentionally only attach a stable 12-char fingerprint and a coarse risk: low/medium/high). The generated profiles are intended to be actionable suggestions, not automatic changes to tenant configs.
Arjun Patel: Left a few nits on the DSL naming; missing docstring on compiler.generate(). Also requested an example mapping for a 99th percentile trigger.
Maya Chen: Addressed naming nits, added docstring and an example mapping in docs/simulation/README.md. Pushed fix in commit d99a7b2.
Lena Ortiz: Concerned about telemetry cardinality growth if we attach raw sketches. Suggest using stable fingerprint only. Also asked for a guard to prevent auto-rollout of suggestions.
Maya Chen: Telemetry now only emits profile_fingerprint and coarse risk_score. Added explicit 'apply=false' default in suggested-profile JSON and documented the manual rollout workflow.
Samir Nair: Noted simulator assumption that token cost is linear — asked for a flag to toggle non-linear cost models for future work.
Maya Chen: Added --cost-model flag and left a TODO to integrate token-tiering pricing curves (ENG-4302).
Arjun Patel: Approve after fixes. Lena Ortiz: Approve. Samir Nair: Approve with comment about later work on pricing curves.
