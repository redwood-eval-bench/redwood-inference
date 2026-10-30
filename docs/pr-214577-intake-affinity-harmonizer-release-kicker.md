# Introduce intake affinity harmonizer and release-kicker for concise changelog fragments

Motivation: Onboarding teams frequently provide heterogeneous metadata (dataset tags, eval buckets, quantization hints) and the registry needs a light, deterministic way to canonize those attributes so downstream profile diffs and release notes are readable and actionable. This PR adds two coordinated pieces: 1) an intake affinity harmonizer that computes a compact affinity signature and canonical attribute map for every model variant during upload; 2) a release-kicker service that consumes profile deltas and emits concise changelog fragments (one-line impact blurb + signal tags) used by the release draft generator.

What this PR does (single-line summary): Adds a deterministic attribute harmonizer at intake and an automated changelog fragment generator that synthesizes concise human-readable notes from profile deltas and affinity changes.

Details / implementation notes: The harmonizer lives under lib/harmonizer and applies a two-pass strategy: normalization rules (case / synonym mapping), then weight-based attribute affinity scoring (dataset, tokenization family, finetune objective). The affinity signature is a short base32 string (aff-...) that captures dominant attributes and a 3-bit mask for notable signals (quantized, low-latency, SSE-enabled). The release-kicker runs as a job in cmd/release and takes delta bundles (profile-previous.yaml, profile-new.yaml) and outputs a compact JSONL stream of fragments; each fragment has a title, terse impact, and suggested severity tag.

API / schema changes: 
- intake manifest adds fields: canonical_attributes (map), affinity_signature (string), affinity_score_vector (CSV). For compatibility we append these fields; older consumers ignore unknown keys.
- release-kicker CLI: `release-kicker --delta <bundle> --out fragments.jsonl --rules ./rules/shortcuts.yaml`

Rollout plan and migration: 
- Step 1 (canary): enable harmonizer for 5% of uploads in redwood-model-registry staging; observe canonical attribute distributions and affinity collisions.
- Step 2: enable release-kicker in dry-run mode to emit fragments to S3; compare against current human-written notes for precision/recall.
- Step 3: enable kicker for auto-draft tagging (but still require human approval before publishing).

Backward compatibility: Fields are additive. The intake CLI falls back to legacy path when harmonizer config missing. No schema migrations required for existing profiles.

Testing and metrics: Added unit tests for the normalization rules, affinity collision detector, and kicker fragment scoring. Local microbenchmarks: harmonizer adds ~14ms median CPU work per upload (measured on a 4-core runner), kicker processes 1k profile deltas in ~9s in the test harness. We added a smoke end-to-end test that uploads a synthetic model and asserts a fragment with expected tags.

Security and privacy: affinity signatures are deterministic but do not include PII. We explicitly strip dataset URIs and only retain high-level dataset tags (e.g., `web-scrape`, `code`, `instruction-tuning`).

Checklist: 
- [x] Harmonizer implementation and CLI integration
- [x] Release-kicker implementation and rule set
- [x] Unit + integration tests
- [x] Documentation: onboarding README and changelog guidelines
- [x] Lightweight rollout plan and monitoring doc added to docs/rollout/harmonizer.md

Why this is new/different: Existing PRs focused on full-blown metadata rewriters or heavy-weight changelog assemblers. This approach emphasizes a minimal, deterministic signature and small concise fragments so that release notes remain readable and machine-combinable without bloating the manifest.

Files of interest (summary): See changed_files for full list. Sample manifest addition: "canonical_attributes": "{\"domain\":\"code\",\"objective\":\"instruction\"}", "affinity_signature": "aff-7KJ3".

Related: ENG-1974 (harmonizer design doc), ENG-2051 (changelog UX tweak), SUP-884 (support case: onboarding noise in release notes).
Marco Liu: Nice direction — the affinity signature looks compact. Question: how do we avoid collisions for large-scale variant sets?
Author (Aisha Patel): We added a collision detector that increments a suffix and emits a warning in the intake log; collisions are rare because the signature encodes a weighted vector and a salt derived from model sha256. We'll add a dashboard alert if collisions rise above 0.1%.
Sana Ortega: Please add an opt-out flag for customers who want zero extra processing on upload (compliance customers may want this).
Author (Aisha Patel): Added `--harmonizer-disabled` flag on the intake CLI and a config toggle. Mentioned in rollout doc.
Elliot Ramsey: Kicker: how do we avoid noisy fragments from tiny perf fluctuations?
Author (Aisha Patel): Kicker uses configurable thresholds: absolute token-latency change, relative perf-percentile change, and a signal-stability score. Defaults are conservative; included rule set in rules/shortcuts.yaml.
Marco Liu: CI flake on test_kicker.py L57 — looks timing sensitive.

Author (Aisha Patel): Fixed by converting to deterministic fixture and bumped timeout. Re-ran CI.
Sana Ortega: Approved pending docs clarifications and rollout plan addition.

Elliot Ramsey: LGTM.
- Add canonical attribute harmonizer to intake; produces affinity_signature for model variants.
- Add release-kicker to auto-generate concise changelog fragments from profile deltas (dry-run first, opt-in).
- CLI flags: --harmonizer-disabled, release-kicker --dry-run.
Notes: No breaking schema changes; fields are additive.
