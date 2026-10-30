# Introduce affordance anchors and release-sprocket for compact changelogs

Motivation: We want to make model onboarding metadata more actionable for runtime and downstream teams and to automate concise changelog generation for model releases. This PR introduces two pieces: (1) affordance_anchors: a small, opt-in annotation block in the model manifest that surfaces recommended serving knobs (preferred quantization profile, batching hints, expected token-distribution tail behavior, and canonical prompt examples) and a confidence/coverage score; (2) release-sprocket: a lightweight changelog assembler that consumes profile deltas, sample-coverage matrices, and affordance anchors to produce compact release notes suitable for console UI and email digests.

What changed (high level): added affordance_anchors schema to manifest v2; migration helper to backfill older manifests with best-effort anchors from profiling traces; a CLI "release-sprocket" tool (Go) that emits a condensed JSON + markdown brief and a one-line summary; unit and integration tests for schema, migration, and sprocket; updated onboarding docs and two example manifests.

Why this is different: existing profile and changelog pipelines focus on performance deltas and semantic diffs. Affordance anchors bridge the gap between profiling heuristics and actionable serving recommendations, and release-sprocket stitches that context into a human-readable, concise note aimed at SREs, product owners, and SDK consumers.

Checklist:
- [x] Manifest schema change and docs
- [x] Migration/backfill helper with dry-run mode
- [x] Sprocket CLI + unit tests
- [x] Integration test that validates compact note generation from a sample delta
- [x] Release notes draft template

Related: ENG-3127 (affordance schema design) and SUP-1421 (customer-requested concise changelogs).
Daniel Park (reviewer): Can we include a TTL for the backfill so stale profiling doesn't override newer telemetry?
Priya Nair (author): Added --max-age flag to the backfill helper; defaults to 90d. Updated tests to validate behaviour.
Sofia Martinez (reviewer): release-sprocket output looks good, but the one-line summary sometimes repeats the same metric. Suggest deduping and preferring higher-impact items.
Priya Nair (author): Adjusted the scoring heuristic in assembler.go to collapse similar metric deltas and prefer throughput/latency regressions over micro-variations in per-token cost. Added a unit test for dedupe logic.
Arjun Rao (reviewer): CI flaked on tests/integration/test_release_sprocket.py on first run (timeout). Re-run after test timeout increase. Approve after green.
CI Bot: initial run failed (integration timeout), re-run requested by @ArjunRao
CI Bot: re-run passed
Sofia Martinez: LGTM
Daniel Park: Approved
Arjun Rao: Approved
- Introduced affordance_anchors in manifest v2: structured serving hints (quantization, batching, prompt exemplars) with coverage score.
- Added migration helper to backfill affordance anchors from profiling traces (dry-run available).
- Added release-sprocket CLI: generates compact changelogs that combine profile deltas, sample coverage, and affordance recommendations. Useful for console summaries and release emails.
- Updated docs and onboarding examples to demonstrate affordance usage.
