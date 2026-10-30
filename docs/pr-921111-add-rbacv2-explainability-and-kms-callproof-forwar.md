# Add RBACv2 explainability layer and KMS call-proof audit forwarder

Adds an explainability shim for RBAC v2 decisioning and a hardened audit forwarder that attaches KMS call-proof bundles to outbound audit records. Feature gated and off by default in private deployments; opt-in via feature flag.
Context: Customers and auditors have asked for both (1) better machine-readable explainability for RBACv2 decisions (so operators can understand why a request was allowed/denied without deep diving logs) and (2) stronger binding between KMS usage and audit events for non-repudiation.

This PR implements two coordinated features: an RBACv2 Explainability Layer (explain middleware + compact decision blobs) and a KMS Call-Proof Audit Forwarder (bundles signed call-proofs with audit events when available).

Design summary:
- RBACv2 Explainability Layer: When enabled, the auth middleware emits a compact decision blob for each evaluated request. The blob contains the matched policy rule ids, relevant attribute values (redacted by policy), decision timestamps, and a deterministic decision fingerprint. Blobs are persisted to the audit pipeline and indexed for short-term retrieval (retention controlled by config). Explanations are intentionally terse and signed by the authorization service to prevent tampering.
- KMS Call-Proof Audit Forwarder: For KMS operations routed through the in-house KMS proxy, the proxy now returns a call-proof envelope (signature over request metadata + region claim). The audit forwarder attaches the envelope to the audit event payload and emits a single canonical record to the compliance ingest (and to a separate secure archive bucket if enabled). Forwarding is best-effort and resilient: when call-proofs are unavailable we still emit the audit with a "call-proof-missing" tag to avoid blind spots.

Migration & compatibility: Feature is disabled by default in hosted; enabled in opt-in private installs via config flag (rbac.explainability.enabled, audit.kms_forward.enabled). Existing audit schemas are preserved; new fields are opt-in and live under audit.extras.rbac_explain and audit.extras.kms_proof. No breaking API changes.

Implementation notes:
- New packages: pkg/rbac/explain, pkg/audit/forwarder/kms_proof
- Authorization service signs decision blobs with service key (rotated via existing KMS tooling)
- Forwarder deduplicates call-proofs by request fingerprint to avoid duplicate attachments during retries
- Added a short-lived explainability cache (LRU) to avoid re-evaluating heavy policy trees on retried requests

Tests & CI: Unit tests added for decision blob construction, redaction rules, and call-proof verification. Integration test harness includes an end-to-end flow: request -> auth -> explain blob -> audit forwarder -> mock KMS proxy. CI history: first run failed fmt/lint in pkg/rbac/explain (missing gofmt and docstring), then fixed in subsequent commits.

Security & privacy considerations:
- Explanations redact attributes marked sensitive by data-classifier; developers must add attributes to allowlist when absolutely necessary.
- Call-proofs contain minimal metadata and never include plaintext customer data; proof payloads are signed and stored in the secure archive with access controls.
- Operators can opt-out of storing full call-proof envelopes by setting audit.kms_forward.archive=false.

Rollout plan:
1) Canary behind feature flag for internal accounts (48h)
2) Gradual opt-in for selected enterprise tenants (via console toggle)
3) Wider opt-in and documentation release.

Observability: Added per-route metrics: rbac.explain_emitted, rbac.explain_cache_hit, audit.kms_proof_attached, audit.kms_proof_missing.

Docs & runtime config: Updated docs/ops/security/rbac_explainability.md and docs/ops/security/kms_audit_forwarder.md.

Linear & Jira: ENG-4213 tracks the feature work; SUP-342 captures support runbooks.

Risks & mitigations:
- Performance: explainability adds ~1.5ms median to auth path in microbenchmarks; mitigated by LRU cache and sampling config.
- Storage: decision blobs increase audit volume; mitigated via configurable retention and sampling (rbac.explain_retention, rbac.explain_sample_rate).

Checklist:
- [x] Unit tests
- [x] Integration tests
- [x] Docs
- [x] Config flags
- [x] Security review by SRE
- [x] Manual canary verification

Commits summary, changed files and tests are listed below.
Diego Alvarez: Can you clarify how the redaction allowlist is managed? Is there an admin UI?
Maya Chen: Answered — redaction allowlist is a configmap for private installs and a console-only toggle for hosted; added docs section and an audit event when attributes are allowlisted.
Sandeep Rao: The call-proof envelope format needs a version field for future migration and tool compatibility — please add.
Maya Chen: Added version field (v1) and included schema in docs; updated tests to assert presence of version.
Lena Fischer: Concern: potential audit spam if KMS proxy intermittently fails. Suggest adding a batching window.
Maya Chen: Implemented a 200ms batching window and de-dup logic in archive_writer; added metric audit.kms_proof_batch_dropped.
SRE (auto-comment): load-test showed 1.5ms median auth overhead with 95th at 6ms under synthetic policy; acceptable for our SLOs but documentable.
Diego Alvarez: Approving after Sandeep's changes and perf notes.
