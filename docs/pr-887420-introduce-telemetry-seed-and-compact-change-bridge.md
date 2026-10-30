# Introduce telemetry-driven signal-seed versioning and compact-change bridge for onboarding

Context: onboarding churn has increased where model profile metadata (performance/latency footprints and affinity hints) drift from real traffic signals and release notes are noisy or missing. This PR implements a deterministic 'signal-seed' that derives a reproducible version seed from recent telemetry (latency histograms, sequence-length distribution, hardware affinity scores) and a compact-change bridge that emits concise changelog anchors for onboarding bundles so release notes can be auto-populated with minimal manual edits. Motivation: reduce manual curation time, improve rollouts by tying profile versions to observed signals, and make changelog generation robust to noisy diffs.

What I changed (high level):
- Added src/onboard/signal_seed.py: seed generator that ingests sampled telemetry snapshots and produces a signed seed blob (hash + provenance hints) used to stamp profile versions.
- Added src/changelog/compact_bridge.py: compact changelog synthesizer that consumes profile deltas + seed provenance and emits one-line release anchors and contextual hints for the Console and release drafts.
- CLI: cli/onboard_seed.py and hooks/onboard_release_hook.py to extract telemetry, generate seed, and attach compact changelog fragments to the intake manifest.
- Console integration shim: webapp/static/onboard_changelogs.js (small client to show compact snippets) and backend endpoint api/onboard/changelog_preview.py to render previews.
- Tests: tests/unit/test_signal_seed.py, tests/integration/test_changelog_bridge_end_to_end.py and small fixture telemetry snapshots.
- Docs: docs/onboard/signal-seed.md and docs/onboard/changelog-bridge.md with examples and rollout notes.

Checklist: [x] Unit tests added and passing locally; [x] Integration test covering seed->changelog path; [x] Docs draft included; [x] Linear tickets linked (ENG-2147) and SUP-8921 for support follow-up; [x] No schema changes to public model profile contract.

Backward compatibility & safety: The signal-seed is optional; existing onboarding flows fall back to the previous behavior when a seed is not present. The compact changelog emits suggestions only — it does not auto-commit release notes. Seeds are HMAC-signed using the registry's signing key and include a provenance blob (source window, snapshot id, heuristic version) so downstream processes can verify and reproduce the seed.

Rolling this out: feature-flagged under registry.feature_flags.signal_seed (default off). Plan: enable for 10% of intake pipelines, measure consistency with manual curation for 2 weeks, then gradually increase. Telemetry retention window defaults to last 72 hours (configurable).

Commits summary:
- 11a2b3c add signal_seed core implementation and simple CLI
- 8d9f0e4 add compact changelog bridge and preview endpoint
- b7c3d22 wire CLI hook into intake pipeline + feature flag checks
- f0d1e6a tests: seed serialization, signature verification, and bridge e2e
- 3c2a9b0 docs: onboarding guide and rollouts

Notes for reviewers: pay attention to the seed serialization format (src/onboard/codec.py) and the signature verification (src/onboard/signature.py). The HMAC key access uses existing secrets access patterns (no new perms). The changelog bridge contains heuristics (configurable thresholds) that prefer concise anchors for changes where delta_weight > 0.2. If you think thresholds should be tuned, flag it — I left conservative defaults.

Adds a telemetry-seeded, reproducible versioning helper for model profiles and a compact changelog bridge that synthesizes concise release anchors for model onboarding notes. No customer-facing API changes. Improves automated changelog accuracy and helps tie profile deltas to observed workload signals.
Diego: +1 overall; asked for stricter error handling when telemetry snapshots are malformed. Lina: requested an additional unit test for fallback behavior and asked to document default thresholds. Mikhail: raised a question about HMAC key rotation and suggested adding key-id metadata to the seed blob; author replied: added key-id and rotation comment in signature.py, added missing fallback unit test. CI initially failed on lint (long lines in docs) — fixed in b7c3d22. Final approvals from Diego and Lina; Mikhail approved after key-id patch.
