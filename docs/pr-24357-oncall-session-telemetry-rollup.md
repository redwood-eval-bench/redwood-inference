# oncall-session-telemetry-rollup-and-fast-sampling-cli

Motivation: Diagnosing oncall incidents often requires a coherent slice of telemetry tied to the responder session (logs, traces, recent SLO deltas, related alerts). Teams currently take ad-hoc snapshots which are heavyweight and inconsistent. This PR introduces a lightweight on-call session sampler (daemon + agent hooks), a rollup aggregator that compacts sampled telemetry into a shareable artifact, and a CLI to produce and upload rollups for postmortems and async review.

What this change does: - Adds a background sampler that listens to incident-bus events and maintains a 5-15 minute sliding buffer keyed by oncall session id. Buffer entries include compacted spans, top-N logs, recent KPI deltas, and active alerts metadata. - Implements severity-weighted adaptive sampling: critical incidents increase sampling frequency and window size automatically. - Adds a rollup aggregator service that deduplicates similar traces, applies redaction rules (PII/keys), compresses KV snapshots, and emits a signed artifact (jsonl + manifest) stored in the telemetry bucket. - Provides a small CLI (slo-session) to fetch, filter, and upload rollups, with support for linear/incident links and optional dry-run. - Integrates with incident-bot so runbook attachments can include a session rollup link.

Compatibility and opt-in: The sampler is disabled by default; enable with env SLO_TOOLKIT_SESSION_SAMPLER=true or via console toggle. Storage format v1 is forward-compatible; aggregator emits a manifest with version and schema checksum.

Checklist: - [x] Core sampler implementation and buffer eviction - [x] Aggregator dedup + redaction pipeline - [x] CLI commands: create, list, upload, attach - [x] Unit tests for redaction and sampling heuristics - [x] End-to-end test against a synthetic oncall replay harness - [x] Documentation and quickstart in docs/slo/session-rollup.md - [x] Feature flag gating and metrics exported

Testing performed: unit tests (coverage 84%), e2e replay with 120 synthetic incidents, manual integration with a staging incident-bot instance. Performance: sampler adds ~1.6% CPU on a medium oncall-helper node and uses ~40MB additional heap under sustained load; aggregator compresses typical 5-minute sessions from ~12MB to ~420KB.

Rollout plan: merge to main -> enable in staging for two weeks -> enable in canary (10% of orgs) with feature flag -> full rollout. Monitoring: metric slo_toolkit.session_rollup.error_rate, sampler.buffer_size_gauge, aggregator.compress_ratio.

Rollback: disable feature flag; old incident-bot will ignore rollup attachments. Backwards compatibility preserved for storage retrieval.

Related: builds on earlier alert-snapshot and telemetry-sampler work (see ENG-6200).

Added a low-overhead on-call session sampler and a rollup aggregation service plus a small CLI for exporting session rollups. Backwards-compatible storage format; opt-in via SLO_TOOLKIT_SESSION_SAMPLER=true. Includes privacy redaction and severity-weighted sampling. Notes: any configs in production must be toggled via feature flag before rollout.
Alejandro Ruiz: Can we add a short TTL for rollups in the bucket and a configurable retention policy?
Maya Chen (author): Added retention config in deploy/helm and config/feature_flags.yaml; default TTL is 90 days, configurable via SESS_ROLLUP_TTL_DAYS.
Samir Patel: How do we guarantee we don't capture secrets in KV snapshots? Need clear redaction rules and a denylist.
Maya Chen (author): Implemented denylist-based redaction and added a sanitizer unit test that runs against a secret-injection fixture.
Lena Fischer: The adaptive sampler increases window during critical incidents — can we add a guard to stop unbounded growth if incidents are flapping?
Maya Chen (author): Added max_window_minutes with default 30 and a decay factor to avoid runaway state when alerts flap.
If retention or encryption-at-rest policies differ in customer buckets, set SESS_ROLLUP_BUCKET and enable server-side KMS. Watch for elevated aggregator.compress_ratio < 0.1 which indicates a degenerate session payload. Ensure incident-bot webhook permissions allow attachment creation.
Publish quickstart doc and demo video to internal devportal
Flip feature flag in staging and run 2-week canary
Open follow-up ticket ENG-7299 for queryables over rollup artifacts
Merged to main via squash. Changelog entry added under unreleased/notes.md. Feature gated behind SLO_TOOLKIT_SESSION_SAMPLER and rolled out to staging post-merge.
