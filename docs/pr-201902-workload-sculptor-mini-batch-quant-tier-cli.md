# workload-sculptor-cli-mini-batcher-and-quant-tiering

Motivation: provide a lightweight, engineering-facing analysis and suggester (CLI + runtime hints) that produces per-route batching and quantization tier recommendations using sampled traffic, token distributions, and a simple unit-econ model. This reduces manual tuning for common workloads and enables safe, observable defaults for inference cost controls.

What this PR introduces: single-string CLI 'redwood sculptor analyze' that ingests a small traffic sample or evaluation prompts, produces a ranked list of candidate batch sizes, a compact quant-tier map (A/B/C tiers), and an annotated risk score for each suggestion. Adds a runtime "mini-batcher" hint that prefers many small microbatches for latency-sensitive routes (configurable) and a safe fallback path to previous scheduler behavior. Includes config schema extensions for "quantTiers" and "miniBatchHint" with sensible defaults.

Checklist: - [x] CLI and analysis engine implementation - [x] Runtime scheduler hook and unit tests - [x] Integration tests against smoke model and small synthetic traffic - [x] Docs and quickstart example - [x] Telemetry events for suggested change adoption and regression guardrails - [x] Linting and CI fixed after initial run

Usage summary (single-line examples are included in docs): run 'redwood sculptor analyze --sample ./traffic.json --model gpt-small --route /generate' to get recommendations. Output includes estimated tokens/cost saved, suggested batch size, quant-tier mapping (tier_A,tier_B,tier_C), and a conservative accuracy delta estimate based on eval prompts if provided.

Safety and rollout: suggestions are advisory only; the runtime requires an explicit apply step (or a rollout via Console) to change serving config. The runtime scheduling accepts a boolean guard 'enforceMiniBatchForRoutes' and an on-degradation fallback to model-size-preserving variants (e.g., fallback to float16) for routes that see output-quality regressions.

Implementation notes: the econ model is intentionally simple (token-rate x latency SLO x per-GPU cost) so teams can reason about numbers and extend it. We added telemetry hooks to measure pre/post token cost and accuracy on provided eval prompts so that the Optimize team can later build RL-style policy tuning on top.

Backwards compatibility: schema additions are opt-in; defaults keep current behavior. No DB migrations required; workload profiles are optional local artifacts or stored as an appendix in existing profile store (new key 'sculptor-profiles' but not required at runtime).

Related: ENG-5781 (econ cli kickoff), ENG-5799 (mini-batcher scheduling spec), SUP-2147 (customer asked for safer quant recommendations).
Ibrahim Patel: Nice focused tool — helps bridge the gap between profiling and ops. A few asks: add a telemetry field for "suggestion_confidence", unit-tests for the scheduler when many small concurrent routes exist, and ensure the CLI returns non-zero exit codes on parse errors.
Maya Chen (author): Thanks — added suggestion_confidence to telemetry and a new scheduler test 'TestMiniBatcherHighConcurrency'. CLI now returns structured codes; updated tests.
Sofia Martinez: Please ensure the default quant-tier mapping is conservative (tier_A should not recommend int8 unless model flagged as 'quant-verified'). Also, docs should show the safe-fallback lifecycle.
Maya Chen: Adjusted default mapping: tier_A -> q4_0 by default; introduced 'quantVerified' boolean in suggestions. Docs updated with fallback flow.
Ethan Brooks: CI flagged a lint error in pkg/config/schema_v2.json; once fixed, approve. Also request a smoke e2e in perf-canary that runs the suggester on 10k synthetic requests.
Maya Chen: Fixed linting, added a lightweight canary job in ci/perf-scripts (kept it gated to nightly).
Ibrahim Patel: Approving — tested suggested batch sizes against synthetic traffic and saw 18-26% token cost reduction for tail-insensitive routes with no quality regression in our eval set. Recommend monitor alerts on adoption.
Introduce 'workload-sculptor' CLI to generate per-route batching and quantization suggestions; add 'mini-batcher' runtime hint to improve tail latency for latency-sensitive routes; safe defaults are applied only when explicitly accepted. See docs/econ/workload-sculptor.md for usage and migration guidance.
This is a non-breaking, opt-in feature. Recommend staging rollout: enable CLI and telemetry in staging, run nightly canary, then enable 'enforceMiniBatchForRoutes' flag via Remote Config for a small set of routes. No DB schema changes required.
Expected unit-econ improvements for workloads with skewed token distributions and mix of latency/cost constraints. Early testing shows 15-30% cost improvements on batch-friendly routes; microbatched latency-sensitive routes see stable tail latency with small throughput tradeoffs. Telemetry added to monitor suggestion adoption and quality regressions.
Unit tests covering scheduler behavior, integration smoke test with sample traffic, docs example verified, CI lint fixes applied, telemetry events validated in staging namespace
ENG-5860: build a web UI in Console to preview and apply sculptor suggestions
ENG-5871: extend analysis to consider dedicated capacity pricing for Reserved GPU pools
ENG-5899: automated A/B rollout recipe for applied suggestions
