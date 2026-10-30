# introduce-jurisdiction-aware-kms-adapter-and-audit-preservation

Motivation: customers with strict data residency requirements were observing KMS usage that did not always preserve the tenant-declared jurisdiction header across retries and service boundaries. This PR introduces a lightweight KMS adapter service plus RBAC decision hooks and audit preservation middleware to ensure: 1) KMS requests are pinned to an allowed geozone based on operator and tenant claims, 2) audit metadata (region, operator-assertion, provenance) is carried end-to-end and de-duplicated at ingest, and 3) runtime fallbacks are explicit and observable when region enforcement is relaxed for operational reasons.

What this changes: single continuous change set that touches the adapter service, runtime auth hooks, audit writer, helm chart and tests. Key behaviors: - A new internal service services/kms-adapter that validates KMS target region against RBAC v2 decision; it returns an explicit consent token for callers. - RBAC hook (internal/auth/rbacv2/kms_binding.go) consults the policy engine and annotates the request context with a region-assertion claim. - Audit middleware (pkg/audit/preservation.go) attaches an immutable audit-stamp and preserves jurisdiction-relation headers across async handoffs. - deploy/helm/kms-adapter-chart.yaml adds a low-privilege deployment with resource limits and an opt-in toggle in values.yaml.

Rollout and compatibility: adapter is opt-in via feature flag (redwood.feature.jurisdiction_kms_adapter). Existing code paths continue to call the previous KMS client until a config flip is applied. We included a staged rollout plan in docs/compliance/jurisdiction-kms.md and feature gating on Dedicated/Private deployments only for the first quarter.

Security considerations: RBAC checks are mandatory for adapter-backed requests; a fallback mode allows explicit operator consent (audit-only) and is logged at WARN level. The adapter performs strict signature checks on incoming operator assertions and rejects mismatched territory codes. All KMS credentials remain encrypted in our KMS; the adapter authenticates to KMS using a service principal with a scoped CMK.

Testing: unit tests for RBAC decision mapping; integration tests that simulate multi-region operator-tenant combinations; e2e test which asserts audit header presence in the ingestion pipeline. CI initially had a flaky e2e that failed due to timing in KV cache warmup — that was fixed by increasing retry window and adding a health probe.

Checklist: - [x] design doc linked (ENG-3421) - [x] unit tests - [x] integration tests - [x] helm chart and values - [x] docs added under docs/compliance - [x] rollout plan and feature flag - [x] security review by SRE and Product Security

Notes: This PR is intentionally coarse-grained to keep the adapter and RBAC integration in a single merge to simplify rollout testing. Follow-ups planned: - split adapter to its own repo for large customers - telemetry upgrade to add per-tenant KMS usage cost attribution (ENG-3502)
Adds a jurisdiction-aware KMS adapter and RBAC v2 hooks to enforce region pinning and improve audit header preservation. No breaking public API changes; install notes in docs/compliance/jurisdiction-kms.md.
Aisha Rivera: high level - this looks reasonable. Please add a note describing the intended behavior when RBAC is unavailable (circuit-breaker semantics).
Jonah Patel: replied - added circuit-breaker semantics and config knobs (adapter.timeout_ms, adapter.circuit_threshold) in deploy/helm/values.yaml; documented behavior in docs/compliance/jurisdiction-kms.md.
Markus Li: nit - prefer a shorter default retry window in the e2e to avoid long CI runs; otherwise +1 after tests pass.
Jonah Patel: replied - shortened default retry and added a CI-only override; pushed a fix in a3f7b2c.
Eleanor Chen: security review - confirm that audit stamps are immutable and signed; also request a short design note about how operator consent is recorded.
Jonah Patel: replied - audit stamps use HMAC with rotating key and are stored in the audit envelope; added design note in docs/compliance/jurisdiction-kms.md and implementation in pkg/audit/preservation.go.
Aisha Rivera: approved - LGTM after the minors.
