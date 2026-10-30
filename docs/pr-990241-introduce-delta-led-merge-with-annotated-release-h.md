# Introduce delta-led profile consolidation with telemetry-annotated release heuristics

Motivation: onboarding churn and noisy release changelogs have made it hard for integrators to understand which model profile changes actually matter for production workloads. This PR introduces a delta-led consolidation step that combines incoming profile deltas with historical telemetry (usage-weighted buckets) to produce a single consolidated profile and an annotated release summary. The summary includes expected latency/cost impact, confidence bands, and automatic tags for changelog generation.

What this implements: consolidated merge pipeline (delta-led) that treats incoming profile changes as first-class deltas instead of blind overwrites; a telemetry bridge that ingests recent per-model route histograms and produces weight vectors used by the merger; an annotation layer that synthesizes release-notes snippets with quantitative deltas (median latency change, token-cost estimates, sample-percent-of-traffic impacted) and human-friendly rationale; a safeguard layer that blocks auto-merge when signal is ambiguous (low sample count or high variance) and surfaces a recommended manual review checklist.

User-visible outcomes:
- New endpoint in the registry API: POST /profiles/:id/merge-delta (idempotent) which accepts a delta bundle and returns a consolidated profile + release snippet draft.
- CLI command: redwood-registry profiles merge-delta --file delta.json --telemetry-window 7d --dry-run
- Release automation hook: on successful consolidation the hook can populate draft release notes with the annotated summary and add tags (impact:low/medium/high, recommended:manual_review/auto_merge).

Implementation highlights (high-level commits):
- commit 1: core merger runtime and deterministic ordering for delta application (src/registry/merge/delta_merger.py)
- commit 2: telemetry adapter that normalizes histograms and computes route weights (src/registry/telemetry/adapter.py)
- commit 3: annotated-summary generator and templating infra for release notes (src/registry/release/annotator.py, templates/release/snippet.mustache)
- commit 4: API + CLI wiring and feature flag gating (api/routes/profiles.py, cli/commands/merge_delta.py, feature_flags.yaml)
- commit 5: test harness and synthetic telemetry fixtures (tests/integration/test_delta_merge.py, tests/fixtures/telemetry/*.json)
- commit 6: docs: how-to + guardrails (docs/registry/delta-merge.md)

Files touched (high level):
- Added: src/registry/merge/delta_merger.py, src/registry/telemetry/adapter.py, src/registry/release/annotator.py, cli/commands/merge_delta.py, templates/release/snippet.mustache, tests/integration/test_delta_merge.py, docs/registry/delta-merge.md
- Modified: api/routes/profiles.py, src/registry/profile_schema.py (added delta schema), feature_flags.yaml, package manifests and CI workflow for telemetry tests

Backward compatibility and safety: the feature is off by default behind the feature flag registry.delta_merge.enabled. The POST /profiles/:id/merge-delta endpoint requires an allowance role (registry:merge) and logs a signed proof bundle for auditability. Auto-merge to mainline profile only occurs when sample-weight >= 500 and variance < threshold or when the release annotation explicitly marks low impact. Otherwise the consolidated profile is saved as a draft and a reviewer is requested.

Testing and validation: included unit tests for delta application ordering, property-based checks for idempotency, integration tests using synthetic telemetry (traffic mixes) and a smoke test against a small real-traffic snapshot (privacy-filtered). CI runs lint, unit, integration and a lightweight telemetry validator job that exercises adapter normalization.

Performance: the delta merger operates in O(n log k) where n is number of delta ops and k is number of route buckets; a synthetic benchmark shows 95th percentile merge time for a 1k-delta bundle is ~420ms on our standard ci runner. The telemetry adapter is optimized for streaming histogram fold and uses a small LRU cache for recent route weights.

Open questions / future follow-ups: adding an explainability view that links sample examples to profile changes (ENG-1719), richer cost-model integration with Redwood Optimize, and expanding the telemetry window heuristics to per-customer baselines.

Related work: builds on earlier profile-onboard efforts but differs by making deltas first-class and tying merge decisions to real usage signal; see ENG-1021 for historical context.

Checklist: Run unit tests, add integration fixture, validate feature flag gating, add docs.

Diego Morales: Great direction. Two asks: (1) add a small E2E that runs the whole path including API -> merger -> annotator so we can validate the snippet format; (2) make the telemetry-window configurable via the API call. responded: added e2e smoke and query param telem_window
Jordan Lee: Could we expose the confidence band values as numeric fields in the API output in addition to human text? requested change: include confidence_low/confidence_high floats. responded: added numeric fields and unit tests
Amira Hassan: Security note: ensure the audit bundle doesn't include raw customer tokens. requested change: redact tokens in proof bundle. responded: implemented redaction + added test to verify no sensitive keys are serialised
Reviewer (ops): Please include a small operation runbook for manual review flow. responded: docs/registry/delta-merge.md includes the reviewer checklist and runbook
