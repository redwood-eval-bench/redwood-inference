# Enforce service-level region pin with KMS shard affinity and compact audit sampling

Motivation: customers with strict data residency requirements asked for a deterministic, enforceable mechanism to pin an entire service's requests to a region+KMS shard so operator actions and data never cross jurisdictional boundaries. This PR implements a policy-level 'service region pin' that is enforced by RBAC v2 decision filters, coupled with a KMS shard affinity allocator and a compact audit sampling pipeline to keep evidence bandwidth reasonable while preserving compliance fidelity.

What this PR does (high level):
- Introduces a new RBAC v2 predicate (ServiceRegionPin) that is evaluated early in the policy pipeline and denies cross-region routing for services that opt-in to pinning. Policy lives under services/policy-engine/rbacv2 and is feature-flagged for staged rollout.
- Adds a KMS shard affinity allocator so KMS requests for pinned services are routed to the correct regional shard; allocator supports soft-fallback to a read-only audit-only shard if the primary is degraded.
- Implements an audit sampler with deterministic sampling by tenant cohort and variation for high-risk operations (admin/elevation events are always retained). Sampling reduces storage/ingest costs for long-running tenants while preserving required trails for audits.
- Adds migration tooling and a small migration SQL to backfill existing service metadata with region-pin annotations for services that already declared data residency in metadata.
- Exposes openapi changes and a new console hint for "Service > Deployment > Region Pin" so operators can opt-in during rollout.

Design notes:
- Policy order: ServiceRegionPin short-circuits routing decisions and is designed as a hard deny early in the evaluation chain. It emits a standardized denial reason code (RESIDENCY_PIN_VIOLATION) used by both API and observability hooks.
- KMS affinity: a lightweight shard key is derived from tenant id + service id + declared region. The allocator is conservative and will not perform cross-region promotions unless an explicit graceful-failover policy is set by Dedicated/Private customers.
- Audit sampling: deterministic hashing ensures reproducible sample windows for replay and evidence reconstruction. Admin paths, operator impersonation attempts, and KMS allocation failures are exempt from sampling.

Rollout plan:
- Feature-flag behind flags.security.service_region_pin.
- Canary 1 (internal): enable for 3 internal services in staging for 48h, validate deny codes and KMS affinity logs.
- Canary 2 (beta customers): opt-in for 2 beta tenants on Dedicated, validate audit replays, and measure ingest reduction.
- Global: enable for all new services by default, existing services require explicit opt-in or migration.

Testing and validation:
- Unit tests for policy predicate evaluation, KMS allocator edge cases and hash determinism added.
- Integration test simulates cross-region request and expects RESIDENCY_PIN_VIOLATION. A flaky timing-based integration test was retried in CI and passed on the second run.
- Load bench validated allocator latency impact at 95th pct (sub-ms median change, 2–4ms tail in synthetic tests).

Operational/Docs:
- New operator playbook: docs/compliance/service-region-pin-playbook.md explains opt-in, audit expectations and incident runbook.
- Console hint and API docs updated so customers can see pin status and KMS shard association.

Checklist:
- [x] Design review with security and compliance (ENG-4821)
- [x] Testing (unit + integration)
- [x] Docs + console hint added
- [x] Migration script included
- [x] Feature flag gating and rollout plan included

Risks and mitigations:
- Risk: accidental denial of traffic during rollout. Mitigation: feature flag + canary + explicit migration script.
- Risk: KMS outage causing pinned service failures. Mitigation: read-only audit-only shard fallback and operator-visible degraded state.

Related PRs / context: this complements recent RBAC v2 work and audit compaction efforts. See ENG-4821, SEC-170 for policy acceptance criteria.
Arjun Patel: Please document the exact guarantee of the allocator under failover—what happens on multi-region replica writes?
Maya Chen (author): Allocator will not rewrite primary region; we added explicit text in the playbook and a conservative fallback to read-only audit shard. See migration notes in the PR description.
Leah Gomez: Suggest making the denial code machine-parsable for alerting systems (e.g., use a fixed numeric code).
Maya Chen (author): Added RESIDENCY_PIN_VIOLATION as a stable string and numeric code 4501 in pkg/observability/events/denial_reasons.go. Unit tests added.
Thomas Rinehart: Integration test timing was flaky in CI; increase timeout and retry once. Also add explicit assertion that KMS shard id matches expected region in logs.
Maya Chen (author): Increased timeout and added deterministic log assertion. CI reran and passed.
Arjun Patel: Approving after validation in staging. Left a minor suggestion about telemetry labels which I recommend in a follow-up small PR.
Leah Gomez: Approved. Verified documentation and console hint UX with product.
Merge bot: All required checks passed. PR merged by squash.
- Added Service Region Pin RBAC v2 predicate to enforce service-level residency pinning.
- KMS shard affinity allocator ensures region-local KMS usage for pinned services.
- Deterministic audit sampler reduces ingest while preserving compliance evidence; admin/elevation events always retained.
- Migration script to backfill service metadata and console hint for opt-in.
Adds a compliance control to prevent cross-region routing for pinned services; KMS affinity reduces risk of key exposure across jurisdictions. No new secrets introduced. Review completed by security team (SEC-170).
1) Small telemetry follow-up to add 'region_pin=true' label to latency metrics (Arjun's suggestion). 2) Consider a lightweight operator CLI to bulk-opt services during migration for large orgs. 3) Evaluate long-term storage tiering for sampled audit events.
