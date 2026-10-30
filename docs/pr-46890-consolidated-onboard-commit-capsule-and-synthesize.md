# Consolidate onboard commit-capsule and auto-synthesize release briefs

Motivation: onboarding currently scatters commit-level metadata, perf deltas, and release note fragments across artifacts and the model profile DB. That makes lightweight release summaries hard to generate and forces reviewers to cross-reference multiple sources. This change introduces a compact 'commit-capsule' blob that travels with a model profile (stored as signed metadata alongside the profile artifact) and a small pipeline to auto-synthesize concise release briefs from the capsule + perf deltas.

What this PR does: 1) Adds a commit-capsule format (minimal JSON) that contains: commit SHA, onboarding CLI user, dataset snapshot tag, top-level performance tokens (latency/throughput/embedding-l2), a short human author blurb, and a list of significant calibration flags (quantization/seq-len/kv-cache-behavior). 2) Extends the onboard CLI to emit the capsule and attach it to the profile upload payload. 3) Introduces a release-brief synthesizer service (local transformer-free template engine) that aggregates capsule metadata and latest profile diffs to produce a one-paragraph release brief plus a 3-line perf summary. 4) Migration: a migration job scans recent profiles (90 days) and attempts best-effort capsule backfill from commit messages, upload metadata, and CI artifacts. 5) Console: adds a compact 'release brief' field in the profile summary view and a download link for the capsule.

Checklist: - [x] Define capsule schema and stability guarantee. - [x] Implement CLI emit and upload hooks. - [x] Implement synthesizer service and unit tests. - [x] Add console rendering and download endpoint. - [x] Add migration job and monitoring. - [x] Add Linear tickets and docs.

Backward compatibility: capsule is additive; profiles without capsules remain readable. Synthesizer falls back to a heuristic brief if capsule missing. Signed capsules use existing model-registry signing keys; no KMS changes required.

Rollout plan: merge to main; enable synthesizer feature-flag by default in staging for 48 hours, monitor release_brief_missing_ratio and brief_length regressions. Run migration job incrementally (10% windows) to avoid API spikes.

Related: ENG-1423 (onboard metadata reliability), PM-89 (concise release notes for partner-facing changelogs).

Notes: This PR intentionally keeps the synthesizer lightweight (no heavy NLP models) to ensure predictable output and cost. Future follow-ups can add an optional LLM-based richer summarizer behind a feature flag.
Liam: asked for stronger validation on 'perf_tokens' (now validated as numeric array with stable ordering). Marisol: requested user-facing example for console; added UI example and e2e test. Arjun: requested migration safety guard; added 10%-window throttling and health-check metric 'migration_backfill_rate'. Author replies clarified the fallback heuristics for missing fields and agreed to a follow-up to add schema docs. Two rounds of small fixes and another quick CI run before approval.
