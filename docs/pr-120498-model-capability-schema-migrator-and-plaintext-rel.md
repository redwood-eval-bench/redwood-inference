# Introduce capability-schema-migrator CLI and plaintext release synthesizer

Problem: legacy model records in the registry use a mix of capability annotations (freeform 'capabilities' strings, per-model tags, and previous 'affordances' blobs) which makes it difficult to compute consistent capability profiles, drive capability-based routing, and produce human-friendly release notes. This change introduces two coordinated tools: (1) a CLI-driven capability-schema-migrator that converts existing metadata into the new normalized capability schema (v2) with a dry-run and audit log, and (2) a plaintext release synthesizer that consumes model metadata + small perf deltas and produces concise, human-readable release snippets suitable for release notes and the onboarding flow.

Design / what changed: capability-schema-migrator: a small CLI (cli/migrate_capability_schema.py) that loads a migration plan (migrations/2025-09-capability-migration.yaml), applies deterministic mapping heuristics (string normalization, alias resolution, confidence scoring), and produces an audit bundle (jsonl) recording original -> normalized mappings and suggested manual review rows. The migrator runs in dry-run mode by default; a --apply flag performs batched updates using the registry API with an optimistic concurrency check. Migration is idempotent and writes a provenance fingerprint onto each updated model profile.

Plaintext release synthesizer: scripts/plaintext_release_synth.py reads the updated metadata and example telemetry items (latency, median token cost, eval delta on prompt-suite) and produces short English release blurbs (two-line summary + one bullet) using rule-backed templates with safe fallbacks for missing measures. The synthesizer is intentionally conservative: it flags records where no reliable perf delta is available and outputs a draft banner instead of a definitive blurb.

Rollback and safety: migration enforces a feature flag (registry.features.capability_v2_migration). If the flag is not enabled, the CLI exits. Each update includes a stamped migration-id and a recorded before/after snapshot in the migration audit store to enable automated rollback. The PR adds an admin page link in the docs showing how to review pending migration bundles.

Testing and verification: added unit tests for alias resolution, edge-case ambiguous mappings, and concurrency retries. CI includes an integration smoke test that runs the migrator in dry-run mode against a snapshot test dataset. Security scan checks that no secret values are written to audit bundles.

Rollout plan: (1) land this PR behind feature flag, (2) run dry-run for full registry and review top-500 low-confidence mappings via a generated CSV, (3) enable --apply in a small cohort (sampled models from partner projects), (4) enable synthesizer to populate a release-drafts queue in the curation UI. We will keep the legacy capability field readable until Q1 2026 and then deprecate it with a migration window notice.

Why this is different: this PR focuses on the metadata migration surface and an editorial-quality, plaintext release synthesizer that is lightweight and safe for product teams to consume. Existing work focused on changelog assembly and compact release notes; this adds a conservative, provenance-forward migration path plus natural-language friendly snippets aimed at reducing manual drafting load.
Diego Alvarez: Please add a dry-run CSV export with confidence scores so SRE and partner teams can triage ambiguous mappings. Author: Added CSV export and included it in the migration audit package; see tests/integration and docs.
Ellen Park: We need a feature-flag gating and explicit migration-id. Also add a unit test that simulates a concurrent update to ensure optimistic concurrency works. Author: Added registry.features.capability_v2_migration gating plus tests that simulate version mismatch and retry loop.
Samir Gupta: Styling nit - prefer shorter template sentences for release snippets to avoid truncation in the Console UI. Author: Rewrote templates to 140-char summary + one bullet; also added fallback 'draft' banner when data is insufficient.
Ellen Park: Security scan flagged potential PII in audit bundles. Please ensure sensitive fields are redacted before bundle write. Author: Audit writer now redacts known PII keys and records a redaction manifest in the bundle metadata.
Diego Alvarez: Approve pending CI green and one small linter fix. Author: Fixed linter and reran CI.
Samir Gupta: Approving — migration path and rollback look solid. 
Added a capability schema migrator CLI (dry-run + apply) and a plaintext release synthesizer to auto-generate concise release note snippets from model metadata and small perf deltas. Migration is gated behind a feature flag and includes audit bundles for review and rollback. Admin docs and CSV export included.
Step 1: Run dry-run and review top-500 low-confidence mappings via generated CSV
Step 2: Pilot --apply on sampled models from partner projects for 48h
Step 3: Monitor migration audit store for redaction warnings and optimistic-concurrency retries
Step 4: Enable plaintext synthesizer to populate release-drafts queue in curation UI
No runtime API surface change. Legacy capability fields remain readable. Migration writes capability_v2 and provenance metadata alongside existing fields.
Model X -> capability_v2: supports('chat', 'structured-output') | confidence 0.98 -> Release blurb: 'Improved chat fidelity and structured outputs; median latency unchanged.'
Model Y -> ambiguous mapping (NLP.task vs 'text-generation') | confidence 0.42 -> Release blurb: 'Draft: capability assignment requires manual review — see migration CSV.'
This PR is intentionally conservative: all applied updates include a migration-id and before/after snapshot to enable automated rollback. The team coordinating the migration will be listed in the rollout doc and a follow-up PR will add a UI view for pending migration bundles.
