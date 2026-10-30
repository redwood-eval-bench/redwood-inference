# Introduce RBACv2 temporal boundaries and KMS audit hasher

Context: Customers operating under strict residency and audit constraints asked for short-lived operator grants and for KMS usage telemetry that reduces exposure of raw key identifiers. This change implements RBACv2 temporal grant semantics (start_time, expiry, max-lifetime) at the policy layer, a deterministic KMS audit hasher that salts and hashes key identifiers before emitting to metrics and long-term logs, and a compiler pass to validate temporal constraints at policy resolution time. Included is a migration helper that maps legacy service grants to expiring equivalents requiring explicit operator consent for lifetime extensions. Checklist: design doc ENG-6128 linked, unit tests added, integration test harness updated, ops docs added at docs/ops/rbacv2-timebound.md, security review completed. Rollout plan: staged canary (1% -> 10% -> 100%) with a feature flag to disable temporal enforcement per region. Impact: no public API contract changes for normal RBAC lookups, operator CLI & dashboard changes required; see release_notes for steps.
Adds RBACv2 timebound grants, KMS audit hashing middleware, and policy compiler temporal checks. Operators must upgrade operator-tooling; see docs/ops/rbacv2-timebound.md for migration steps.
Priya Rao: Can you clarify how the deterministic salt is derived and rotated? Concerned about correlation across tenants.
Maya Chen: Salt derivation is tenant-scoped and seeded from tenant salt material stored in KMS-backed secrets; rotation is supported via migration helper and documented in ops doc.
Jared O'Sullivan: We observed one intermittent gosec failure on CI for use of crypto/rand fallback; include deterministic fallback test and rationale.
Maya Chen: addressed in commit e81a62d, test added and noted in commit message. Re-ran CI and gosec passed.
Liam Nguyen: Approval pending small nits in docs and a request to add a post-deploy smoke test that validates temporal enforcement in canary.
Maya Chen: Added smoke test to integration harness and included canary verification steps in rollout section of docs. Approving now.
Priya Rao: Approving — security review complete, no issues blocking.
Jared O'Sullivan: Approving — looks good to merge with squash.
