# profile-certification-and-compact-release-notes-automation

Motivation: We need a lightweight, machine-readable certification stamp attached to model profiles that permits gating, compliance queries, and front-line QA checks during onboarding. At the same time teams asked for a compact, single-line release blurb for quick release notes rather than the verbatim profile delta output. This PR implements: 1) profile.certification schema (issuer, level, timestamp, evidence_url) + migration logic; 2) validator/schema bump to v1.3; 3) onboarding pipeline hook that runs certification checks and annotates profiles; 4) compact-release-notes generator that consumes profile deltas and emits one-line release blurbs (suitable for changelogs and release headers); 5) docs + tests.

Implementation notes:
- Schema: added `certification` optional object to profile.yaml schema; validator will accept missing certification for legacy profiles.
- Onboard hook: new step `certify-and-annotate` runs after profiling and before publishing; it attaches the certification object when automated checks pass or copies over manual cert fields if present.
- Compact notes: a small assembly utility that maps delta types (profile metadata, perf change, API-compat) into a terse blurb: e.g. "profile: llm-x: perf +5% (fp16) · certified by infra-ci".
- Backwards compatibility: validator has a compatibility mode and a migration script `scripts/migrate-profiles-to-v1.3` that adds empty certification fields when missing to preserve deterministic diffs.
- Observability: metrics emitted for certification failures and compact-note generation counts.

Checklist: [x] schema change; [x] validator update; [x] onboarding hook; [x] compact-release-notes generator; [x] unit + integration tests; [x] docs; [x] migration script; [x] CI integration.

Commits summary:
- Add profile.certification schema and schema v1.3 bump
- Implement certify-and-annotate onboarding hook
- Introduce compact-release-notes assembler and templates
- Add migrate-profiles-to-v1.3 script
- Update validator and add tests
- Docs: onboarding -> certification; changelog examples

Tests: unit tests for schema and assembler; integration test in eval-harness that runs a sample profile through the new onboarding flow; added CI matrix combo to exercise legacy -> migrated profiles.

Files changed (high level):
- registry/schema/profile_schema_v1.3.yaml : new certification stanza and docs comment
- registry/validator/validator.py : support for optional certification and compatibility mode (small snippet added)
- registry/onboard/hooks/certify.py : main hook that runs checks and writes annotation
- registry/tools/compact_notes/assembler.py : generator for compact release blurbs and templates
- scripts/migrate-profiles-to-v1.3 : idempotent migration helper
- docs/onboarding/certification.md : user-facing doc and examples
- tests/unit/test_certification.py, tests/integration/test_onboard_certify.py

Small illustrative snippet (validator compatibility helper): `if 'certification' not in profile and compatibility_mode: profile['certification'] = { 'issuer': null, 'level': 'unverified' }`

Review conversation:
- Jordan Patel (review): "Schema bump looks good; please ensure migration script is idempotent and add explicit unit test for empty-evidence_url handling." 
- Elena Martinez (author): "Added idempotent check and test; clarified evidence_url optional semantics in schema docs." 
- Sofia Ruiz (review): "Compact blurb format is handy; can we ensure the assembler output is stable across runs (deterministic sort)? I saw one nondeterministic ordering in a local run." 
- Elena Martinez: "Assembler now sorts keys and uses stable templates; added test to assert deterministic output." 
- Arjun Kapoor (approval): "LGTM after minor nit. Verified small integration run on staging with the sample 'gpt-variant-3' profile."

CI/Checks:
- unit (pytest): pass
- integration (onboard-e2e): pass after one flaky network test was retried
- lint (ruff/black): pass
- docs build: pass

Merge: Squash merged to main by Elena Martinez on 2026-03-02. Notes: This adds a new schema version (v1.3) but keeps compatibility-mode for older profiles; operators running private deployments should run the migration script before bulk publishes to avoid noisy diffs.

Follow-ups:
- ENG-1297: add manual certification UI in console (not in this PR)
- ENG-1302: add signer verification for evidence_url attachments
Adds a profile.certification block to onboarding profiles; introduces an automated compact-release-notes generator that summarizes profile deltas into one-line release blurbs for model registry releases; updates the validator to accept the certification stanza and migrates existing profiles transparently; no customer-facing API changes.
Jordan Patel: Schema bump looks good; please ensure migration script is idempotent and add explicit unit test for empty-evidence_url handling.
Elena Martinez: Added idempotent check and test; clarified evidence_url optional semantics in schema docs.
Sofia Ruiz: Compact blurb format is handy; can we ensure the assembler output is stable across runs? saw nondeterministic ordering in local run.
Elena Martinez: Assembler now sorts keys and uses stable templates; added test to assert deterministic output.
Arjun Kapoor: LGTM after minor nit. Verified small integration run on staging with sample profile.
