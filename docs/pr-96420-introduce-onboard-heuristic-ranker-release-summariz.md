# Introduce onboarding heuristic ranker and summarized release creator

Context and motivation: model intake currently produces raw metadata and full performance profiles but lacks a lightweight, deterministic ranking that helps reviewers prioritize which model versions to certify first. When dozens of candidate variants appear (quantized, distilled, quant+distill), reviewers need a concise, evidence-backed suggestion plus an initial release blurb. This PR adds a small, pluggable heuristic ranker and a summarized release-note generator that drafts short changelog entries tied to onboarding artifacts.

Design: The ranker assigns a score computed from three weighted signals: stability_score (based on rollout probe pass rate), coverage_score (how many canonical workloads the model hit during preflight), and cost_estimate (tokens/sec and quantization hint). We expose a config object for weights so platform teams can tune heuristics per workspace. The summarizer consumes ranked artifacts and: - extracts the top-3 contributing changes (e.g. quantization, sequence-length tuning, prompt-template hints), - generates a 2-3 sentence release blurb with deterministic templates, - attaches a small Impact badge (latency delta and cost delta). The summarizer is intentionally conservative: drafts are marked "DRAFT: auto" and require human sign-off in the console.

What changed (summary of commits): commit 1: add ranker module and unit tests; commit 2: add summarizer and integration tests; commit 3: add CLI preview tool and docs; commit 4: wire into release assembler and CI workflow; commit 5: fix edge-case where coverage_score==0 and add guard rails.

Files / key diffs: registry/intake/ranker.py  (new) - core Ranker class with compute_score(metadata, perf_profile) -> float. Sample snippet: def compute_score(meta, perf):
    stability = min(meta.get('probe_pass_rate', 0.0), 1.0)
    coverage = perf.get('workload_coverage', 0) / float(MAX_WORKLOADS)
    cost = max(1e-6, perf.get('estimated_cost_per_1k_tokens', 0.0))
    score = weights['stability'] * stability + weights['coverage'] * coverage - weights['cost'] * math.log(cost + 1.0)
    return round(score, 4)

registry/release/summarizer.py (new) - small template engine with deterministic slot filling and a safety layer preventing PII or long free-text passthrough. CLI: cli/onboard_preview.py (new) - preview a model's ranked entry and summarizer blurb locally before pushing. Docs: docs/onboarding/heuristics.md - explains config knobs and when to tune weights. CI: workflow/ci.yml updated to run summarizer integration tests in the release stage.

Testing and CI: Unit tests for ranker cover corner cases (zero coverage, absent probe_metrics). Integration test simulates a small intake pipeline with two candidate variants and asserts the summarizer produces consistent drafts. CI initially failed on the e2e integration due to a mock secrets race; I added a retry in the test harness and updated the workflow to use isolated ephemeral buckets for test artifacts. Final CI status: all checks passed (lint, unit, integration, e2e).

Review conversation (high-level single-string aggregation): Marco Nguyen: "Nice incremental approach. Two asks: (1) make weight config pluggable per-organization; (2) add guardrail to avoid creating a release blurb for variants with <50% probe coverage."
Aisha (author): "Both addressed: added org-level config and a coverage threshold check in summarizer that tags drafts as 'insufficient telemetry' if below threshold."
Dana Sokol: "Can we include a concise sample of the generated blurb in the docs?"
Aisha: "Added examples in docs/onboarding/heuristics.md and the CLI prints a preview sample."
Priya Raman: "Unit tests look good. Please add a changelog entry for the release assembler wiring."
Aisha: "Added a small CHANGELOG fragment and a README note; moved the release assembler step to a named hook so downstream consumers can opt-in."

Backward-compatibility and risks: There are no changes to public REST APIs. The ranker and summarizer run as part of the internal intake pipeline; teams who relied on the previous purely manual review flow will now see auto-drafts — they must opt-in to auto-publishing. The main risk is over-reliance on heuristic scores; mitigations: conservative defaults, prominently labelled drafts, and per-org weight overrides.

Operational notes: Added a feature flag onboarding.heuristic_ranker.enabled=false by default; rollout plan: enable for 25% of orgs w/ high-throughput onboarding for two weeks, gather feedback, then increase. Metrics added: onboarding.rank_score histogram, summarizer.draft_generated_count, summarizer.draft_signoff_rate. Alerts: summarizer.error_rate > 1% triggers pager.

Follow-ups (future work): (1) add ML-backed ranker trained on historical signoffs; (2) allow customizable natural-language templates per org; (3) surface draft signoff analytics in the Console.

Merge notes: this PR was squashed on merge, changelog fragment included in commit message. Merge method: squash and merge.
Adds a heuristic scoring stage to the model intake pipeline that ranks candidate onboarding artifacts by stability, telemetry coverage, and estimated cost-per-token. Produces condensed release note drafts per model version and wires them into the release assembler. No public API changes.

Highlights: heuristic scoring config (pluggable), summarized changelog autodrafts, small CLI to preview release blurbs, unit + integration tests.
