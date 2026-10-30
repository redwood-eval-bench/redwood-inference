# Implement RBACv2 derivative roles, KMS fidelity checks, and residency circuit

Context and motivation: Customers operating in regulated jurisdictions requested stronger guarantees that operator actions cannot cause cross-region key usage and that delegated operator privileges are minimal and time-limited. This PR implements three coordinated features: 1) RBACv2 derivative roles (a lightweight delegation primitive that issues ephemeral 'role leases' scoped to claim subsets and TTLs), 2) KMS fidelity checks (enforce key region, active rotation status, and provider provenance before authorizing use), and 3) a residency circuit that stops requests if KMS usage would violate tenant region policy. Additionally, to keep compliance teams happy, audit telemetry was enhanced with an opt-in masking pipeline that strips PII while retaining cryptographic integrity hashes and provenance metadata.

What changed (high-level):
- Introduced a new RBAC artifact 'DerivativeRole' and an issuing controller in authorization-service. Derivative roles are stored as short-lived claims and can only grant actions explicitly allowed by the parent role plus a constrained scope.
- Implemented 'role-lease' API surface used by services (runtime, operator tools) to request ephemeral delegated credentials. Leases include expiry, allowed-operations, region constraints, and an operator-assertion token.
- Added KMS fidelity verifier: a pluggable check that validates requested key metadata (region tag, rotation timestamp within policy window, provider id) via a KMS metadata cache before signing/wrapping operations.
- Residency circuit: a policy evaluation layer that short-circuits requests when evaluated policies indicate the operation would move data or encryption keys outside authorized geozones. The circuit supports soft-fail (warning + audit) and hard-fail (deny) modes per-tenant.
- Audit changes: audit-writer now supports a masking pipeline that can be configured per-tenant. Masking removes known PII patterns (email, SSN-like tokens) and replaces them with stable pseudonyms; original integrity is asserted via a separate HMAC signature stored outside the body so compliance can prove no tampering.
- Documentation and SDK: python/ts SDKs updated with helper methods for requesting role leases and handling residency rejections. Console docs page added for RBAC delegation flows and tenancy controls.

Commits summary (high-level):
- feat(auth): add DerivativeRole CRD and issuing controller
- feat(auth): role-lease API + token format docs
- feat(kms): add fidelity verifier + metadata cache with TTL
- feat(policy): residency-circuit middleware + tenant config hooks
- feat(audit): implement masking pipeline and integrity hash envelope
- docs: add RBAC delegation and residency pages
- sdk(python,ts): add helpers and examples for role leases
- test: end-to-end tests for cross-region enforcement and audit masking

Files changed (representative):
- authorization-service/cmd/controller/derivative_role_controller.go
- authorization-service/api/v1/derivative_role.proto
- authorization-service/internal/lease/lease_issuer.go
- kms/validator/fidelity_verifier.go
- kms/validator/cache.go
- policy/middleware/residency_circuit.go
- audit/writer/masking_pipeline.go
- docs/security/rbac-delegation.md
- sdk/python/redwood/leases.py
- sdk/ts/src/leases.ts

Testing & CI: Initial CI run failed on lint (ts-sdk lint rule flagged casing in example), author pushed fix and retriggered. Integration pipeline ran full e2e against a local-kms emulator; tests cover: lease expiry enforcement, KMS fidelity rejection for mismatched region or stale rotation, residency circuit hard-fail vs soft-fail behavior, and audit masking performing reversible pseudonym mapping for test keys. Performance test: the fidelity cache keeps median metadata lookup latency <2ms under 2000 qps in the canary harness.

Review conversation (summary):
- Marcus Liu: requested stronger invariants on derivative role issuance (proof attached). Comment: "We should assert parent role last-used timestamp to prevent re-delegation of stale privileges." Author: "Added check in lease_issuer.go to refuse issuance when parent role has been inactive >30d; exposed tenant config to override."
- Priya Singh: raised concern about audit masking interfering with legal discovery. Comment: "Make sure masked values are reversible for audit admins with proper permissions." Author: "Masking pipeline stores HMAC and reversible pseudonym map in tenant-scoped secret store; added docs and tests. Access gated by policy and audit trail."
- Daniel Ortega (SRE): requested a circuit breaker metric and a graceful fallback to cached allowlist for sudden KMS-timeout scenarios. Author: "Added metrics: residency_circuit.denied_count, fidelity_verifier.cache_hit_rate, fidelity_verifier.remote_timeouts; added fallback to cached 'allow' for configured tenants to avoid outages (opt-in)."
- Security review (automated): SAST flagged a weak HMAC key rotation handling in audit writer; author rotated to use tenant-KMS-protected HMAC keys and added rotation test.

Backward compatibility & rollout plan:
- RBAC DerivativeRole is opt-in; controllers ignore CRDs unless enabled via feature flag (rbacv2-derivative=true).
- Residency circuit disabled by default; tenant admins can enable per-organization in Console (soft-fail mode recommended initially).
- KMS fidelity verifier registered as an optional middleware; for customers using third-party KMS providers without region metadata, the verifier falls back to provider-specific heuristics and emits a 'fidelity_unknown' audit event.

Monitoring & alerting:
- New alert: residency-circuit.denied_count > 5% of requests over 5m for any tenant => page SRE.
- New dashboards: KMS fidelity verifier hit-rate, average latency, residency circuit rejection heatmap by tenant/region, audit masking throughput and error rate.

Risks and mitigations:
- False positives: residency policy could block legitimate operations. Mitigation: soft-fail initially, strict whitelisting and clear tenant override path.
- Performance: extra KMS metadata checks add latency. Mitigation: local fidelity cache with TTL + async refresh. Performance tests indicate acceptable overhead.
- Legal discovery: masking must be reversible for authorized roles. Mitigation: pseudonym maps stored in tenant secret stores with access controlled and fully audited.

Follow-ups / TODOs:
- Add UI flows for creating DerivativeRole templates in Console (ENG-5410).
- Add support for multi-provider KMS provenance chaining (ENG-5416).
- Run limited pilot with 3 enterprise customers and collect policy exceptions for two weeks before wide rollout.

Merge outcome: squash-merged into main after approvals from Marcus, Priya, and Daniel and green CI. Changelog entry added under unreleased security features.

Introduces derivative RBAC role support with timebound leases, KMS fidelity enforcement (region + rotation checks), and a residency circuit to prevent cross-zone key usage; audit masking preserves non-repudiation while removing PII.
Marcus: Please assert parent role last-used timestamp before issuance; potential replay risk.
Priya: Masking must be reversible for compliance - how are keys managed?
Daniel: Add metrics and fallback on KMS timeout, else risk outage.
Security SAST: HMAC key rotation missing in audit writer - recommend KMS-protected keys.
Feature-flag gated for safe rollout
Pilot plan recommended before tenant-wide enablement
Observed fidelity cache median latency <2ms at 2k qps in canary
