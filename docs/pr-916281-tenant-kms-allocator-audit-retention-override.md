# introduce-tenant-kms-allocator-and-audit-retention-override

Context and motivation: Customers with strict residency contracts asked for stronger guarantees that encryption keys and audit retention align with tenant residency and contract-level retention windows. This PR introduces a tenant KMS allocator service that: maps tenant residency metadata to a preferred KMS provider, provides deterministic fallback rules, and surfaces an audit-retention-override flag that the contract/compliance team can set to extend/shorten audit retention for that tenant. The allocator is intentionally conservative: it does not create keys, only selects providers and emits a signed allocation assertion that other services can consume.

What this change does (high level):
- New microservice: tenancy-kms-allocator (Go) with a small HTTP API used by provisioning and by the front-line key-wrapping proxy.
- Residency-aware selection rules: configurable region -> provider mapping plus a per-tenant override whitelist.
- Allocation assertion: signed JWT-like envelope (auditable) that records chosen provider, selection rationale (policy id), and TTL for the allocation.
- Audit retention control: a new tenant attribute `audit_retention_override` that the allocator writes as an immutable audit marker. Downstream ingestion honors this marker to extend retention.
- RBAC v2 changes: add `allocator.manager` role (operator-scoped) and policy to disallow tenant-scoped changes by non-operators.
- Migration: backfill job (idempotent) to tag tenants with region hints and to populate `allocator_metadata` for long-tail tenants.

Files changed (summary):
- services/tenancy-kms-allocator/main.go (new)
- services/tenancy-kms-allocator/policy/selector.go (new)
- services/tenancy-kms-allocator/api/handlers.go (new)
- pkg/rbac/v2/policies/allocator_policy.yaml (updated)
- infra/migrations/20260204_tenant_allocator_backfill.go (new)
- docs/compliance/tenant-kms-allocator.md (new)
- internal/audit/ingestor/retain_override.go (updated)
- tests/e2e/allocator_integration_test.go (new)

Commits (summary):
- 1a2b3c4: scaffold tenancy-kms-allocator service, API and CI job
- 2b3c4d5: implement residency selection rules, provider fallbacks, and tests
- 3c4d5e6: add signed allocation assertion generation and verification helpers
- 4d5e6f7: RBAC v2 policy + migration backfill job + docs
- 5e6f7g8: fix flaky e2e test, add stricter mocks for KMS provider responses

Testing and CI:
- Unit test coverage: allocator selector 92%, assertion signing helpers 88%.
- E2E: `allocator_integration_test` exercised a local KMS stub and the audit ingestion pipeline. The first CI run failed on the e2e due to timing assumptions for the audit ingestion consumer (flake). I reworked the test to wait for the ingestion checkpoint instead of sleep; rerun passed. CI checks: build (pass), unit (pass), integration/e2e (pass), lint (pass).

Security review:
- Threat model notes added in docs/compliance/tenant-kms-allocator.md.
- The allocator signs allocation assertions using the platform's signing key; assertions are audited and immutable once emitted.
- RBAC defaults to deny for allocator write paths; only `allocator.manager` and the provisioning service principal can create allocations.

Migration plan:
1) Deploy service in canary mode (read-only) and run backfill in dry-run to gather mismatches.
2) Run migration (20260204_tenant_allocator_backfill) during maintenance window; job idempotent and emits a migration report to S3.
3) Flip allocator to active mode and enable downstream honor of `audit_retention_override` via feature flag `allocator.audithonors.override` for 72h.
4) Rollout RBAC policy two-phase: shadow -> enforce.

Manual testing steps included in PR: how to run local stub KMS, run migration against sandbox, and example curl to retrieve signed allocation assertion.

Review conversation (high level):
- Marco Li (reviewer): "Recommend tightening assertion TTL semantics and make the signing key rotation-friendly (include key id). Also, the selector needs an explicit deny list for prohibited providers per region."
- Author (Priya): "Addressed: added `kid` to the assertion header and made selector consult `config/prohibited_providers.yaml` with unit tests; added a test covering a prohibited-provider veto path."
- Ana Gutierrez: "Can we avoid changing audit retention semantics unless there's legal sign-off?"
- Priya: "We added an `audit_retention_override_request` audit record queue for legal review; the feature is gated behind a feature flag until compliance signs off."
- Ben Thompson: "Migration looks heavy; please ensure the backfill job can be cancelled and resumed safely."
- Priya: "Backfill is checkpointed per-tenant and supports resume/cancel; added tests and a status endpoint."
- Final approvals: Marco Li (approved), Ana Gutierrez (approved after compliance gating), Ben Thompson (approved).
- Add tenant-scoped KMS allocator service with residency-aware provider selection.
- Introduce audit retention override for tenant contracts with audit-immutable tag preservation.
- Add RBAC v2 policy and migration to scope allocator management to operator roles.
- Backfill job to annotate existing tenant records with residency/resolver hints.
- No runtime API changes for customer-facing inference endpoints.
Marco Li: tighten TTL semantics, include kid on assertions
Ana Gutierrez: gate retention changes until legal sign-off
Ben Thompson: ensure migration cancellable and resumable
