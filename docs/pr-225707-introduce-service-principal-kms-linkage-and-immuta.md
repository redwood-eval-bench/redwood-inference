# introduce-service-principal-kms-linkage-and-immutable-audit-proxy

Problem: Customers with regulated workloads require stronger mapping between service principals and KMS keys so that keys cannot be used outside intended service boundaries, and audit trails must be tamper-evident to satisfy compliance reviews. Existing tenant-scoped KMS checks are coarse-grained and do not assert ephemeral delegated callers at the service-principal level.

This PR implements two coordinated capabilities: (1) Service Principal -> KMS Linkage: a policy and runtime enforcement layer that binds KMS key usage to a validated service principal fingerprint (derived from signed token claims and optional org-level allowlists). The enforcer rejects key operations when the principal-fingerprint does not match the binding. (2) Immutable Audit Proxy: a lightweight write-through proxy that signs and anchors audit events to an append-only ledger (opaque blob store + chained signatures) to provide tamper-evidence and a stable export path for compliance teams.

Scope of changes: policy compiler updates (RBAC v2 extension for principal-fingerprints), runtime enforcement hooks in the kms-allocator and signer, new audit-proxy service, migration tooling to backfill existing principal bindings, and relevant docs and e2e tests.

Checklist:
- [x] RBAC v2 policy schema extension and compiler: added principal_fingerprint predicate and validation rules
- [x] Runtime enforcement in kms-allocation path with fallback behavior for unbound keys (deny-by-default)
- [x] Immutable audit proxy service with signing and ledger writer + export CLI
- [x] Migration script to populate bindings from service roster and manual allowlists
- [x] SDK hooks (Python/TS) to surface rejection reasons and audit export utility
- [x] Unit and integration tests, and local e2e test harness for private deployments

Why this matters: Moves KMS scoping from tenant-level assertions to caller-level assertions for service principals, reducing risk of cross-service key use and simplifying compliance audits. The immutable audit proxy creates a defensible evidence trail that is easier to ingest into 3rd-party compliance tooling.

Rollout plan: feature-flagged behind `service_principal_kms_linkage` and `immutable_audit_proxy` settings. Initial rollout in non-production regions for 2 weeks, then staged to Dedicated customers who opt-in, followed by Private packages in patch release. Migration tooling has a dry-run mode and a reconcile job that surfaces ambiguous bindings for operator review.
Marina: Requested stronger error codes for operator readability; asked for structured rejection reasons, not free-text.
Aisha: Updated policy to return rejection codes RFC style and added mapping to SDK helpers.
Jon: Concern about ledger storage growth; asked for retention + compaction plan.
Aisha: Added compaction policy (configurable) and periodic snapshot+vacuum job; documented costs.
Rajesh: Asked for fallback behavior for unbound keys in legacy tenants.
Aisha: Implemented deny-by-default and a `legacy-unbound-allowlist` feature flag with operator approval required for migration.
Marina: Approved after last changes. Jon: Approved after storage/compaction answers.
Merge queue: squash and merge after CI green and manual canary deploy to staging succeeded.
