# Introduce KMS delegation verifier and immutable audit snapshotting

Context: Customers deploying Private and Dedicated silhouettes asked for stronger guarantees around delegated KMS usage and auditable snapshots for regulatory review. This change introduces a KMS delegation verifier (server-side policy enforcer + offline verifier) that prevents transient operator credentials from being used to access keys outside an allowed tenancy/region envelope, and an immutable audit snapshotting pipeline that creates tamper-evident, redacted audit exports for compliance teams.

What this PR does (high level):
- Adds a runtime middleware that validates KMS request contexts against RBAC v2 delegation assertions and the tenant's declared region pin. Requests that fail verification are rejected with a 403 and a structured denial which includes the policy failure reason code.
- Introduces an offline verifier binary (cmd/kms-delegation-verifier) that can replay request traces and validate that historical delegations complied with policy; useful for audits and post-incident validation.
- Adds an immutable audit snapshotter service that batches and writes signed, chunked snapshots to an object store. Snapshots include a redaction layer (configurable by data classification) and a signature envelope (ed25519) plus a SHA256 ledger file for integrity checks. Customers can configure the destination S3/GCS bucket or opt into Redwood-managed archival.
- Adds RBAC v2 policy templates and a small policy compiler that synthesizes operator-scoped assertions into runtime checks; includes policy for cross-region delegation deny-by-default.
- Adds Linear/Jira links in alerts and a migration guide for customers using older delegation semantics.

Why this approach:
- Runtime protection prevents new delegations from being abused in real-time.
- Offline verifier supports compliance review and forensic replay without touching production servers.
- Snapshotter design intentionally keeps a signed ledger and redact rules outside of customer keys to allow regulatory review while preserving PII minimization.

Rollout plan: staged feature-flag rollout:
1) Dry-run: runtime middleware runs in audit mode only for 48 hours (collects denials but does not enforce).
2) Canary (10%): enforce denies for non-critical tenants.
3) Full: enforce denies for all tenants after 7 days with monitoring and rollback playbook.

Migration notes and config flags:
- New flags: --kms-delegation-enforce=(off|audit|enforce), --audit-snapshot-target=(redwood|customer-bucket), --snapshot-retention-days.
- Policy templates added under deploy/policies/rbacv2/kms_delegation_templates.yaml.

Testing and validation:
- Unit tests added for verifier logic and redaction rules.
- Integration tests added that exercise cross-region KMS attempts (mocked KMS providers).
- End-to-end snapshot integrity test verifies signature + ledger chain.

Risks and mitigations:
- Risk: false positives for complex delegations (mitigated by dry-run and policy compiler warnings).
- Risk: snapshot storage costs (mitigated via retention flags and customer-managed buckets).

Checklist:
- [x] Design doc (Linear ENG-1942) linked and reviewed
- [x] Unit/integration tests added
- [x] Policy templates included
- [x] Monitoring dashboards draft (observability)
- [x] Rollback playbook authored

Related: SUP-887 (support case where a delegated operator key was used from an unexpected region).
Evan Morales (2026-01-31): Please split the middleware validation path so high-latency verification (policy compiler lookups) does not block fast-path KVS ops. Also add metrics for validation latency.
Priya Natarajan (2026-01-31): Split into fast-path (cached assertion lookup) + background policy refresh implemented; added metrics `verifier.latency_ms` and cache hit ratio.
Jonah Li (2026-02-01): The offline verifier binary is great. Can we add an option to produce a CSV report of failures for legal teams?
Priya Natarajan (2026-02-01): Added --export-csv flag and documentation in the migration guide.
Marisol Ruiz (2026-02-02): LGTM after CI re-run. One minor nit about error codes; suggest adding a mapping table to docs.
Marisol Ruiz (2026-02-02): Approved
Security: Add KMS delegation verifier to enforce tenant/region-bound key usage; Compliance: Immutable audit snapshotting with redaction and signed ledger; Ops: New flags for dry-run and snapshot retention.
Default rollout is audit-mode. Operators must ensure the snapshot bucket has write permissions and key rotation for signing keys is coordinated with Security. See docs/ops/kms-delegation-migration.md for playbook.
