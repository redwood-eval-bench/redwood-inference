# Introduce tenant-scoped BGP subnet allocator and GPU nodeclass orchestrator

Context and motivation: Private deployments require strong isolation of network and GPU resources per tenant while keeping Terraform state manageable for multi-tenant customers. Existing TF modules relied on coarse VPC-level subnet allocation which caused contention during concurrent tenant onboarding and complicated rollback paths when network claims failed. This PR implements a tenant-scoped BGP subnet allocator and a GPU nodeclass orchestrator to reduce allocation conflicts, enable safer upgrades, and support tenant-prefixed TFstate layouts for easier recovery.

What this PR does (high level):
- Introduces terraform/modules/tenant-subnet-allocator: a declarative TF module that exposes a 'lease' API for requesting contiguous /24 subnets from a BGP-able address pool. The allocator writes an immutable lease record into a prefixed TFstate key (configurable prefix per-tenant) to prevent double-allocations across apply retries.
- Adds terraform/modules/gpu-nodeclass-orchestrator: decouples tenant leases from nodeclass definitions so GPU nodepools are created once and bound via node affinity policies at tenant claim time. This reduces duplicate nodepool churn and speeds onboarding.
- Implements terraform/modules/iam-shard: a pattern for scoping IAM roles and KMS grants per-tenant using a stable hash-derived namespace to avoid cross-tenant collisions and to make rotation predictable.
- Adds terraform/modules/state-prefix-migrator: a small CLI wrapper + script (migrate.sh) to perform safe TFstate prefixing/migration when customers opt into tenant-prefixed states. Migration performs a dry-run, copies state objects to the new prefix, and leaves tombstone markers to allow rollback.
- Installer changes: helm chart values.yaml gains new knobs for allocator settings and rollout strategy; upgrade-hooks/plan-fence.sh and rollback/rollback-checker.go add fences to block simultaneous plan+apply workflows for overlapped tenants.
- Tests: integration tests added under tests/integration verifying concurrent tenant claims (simulated BGP pool exhaustion), and nodeclass binding behavior.

Design notes and trade-offs:
- Lease model: the allocator uses an optimistic lock in TFstate (lease record + index) to avoid introducing an external database dependency. This keeps the change self-contained in Terraform but requires careful rollback semantics (handled by the state-prefix-migrator and plan-fence hooks).
- BGP behavior: module exposes hook points for route-programmers; default behavior emits route objects compatible with the on-prem router shim used by the installer. Customers with external routing systems can provide a 'route_operator' null provider.
- IAM sharding: we intentionally use a deterministic hash to allow stable role lookup by operator tooling; the module also emits a role rotation helper for scheduled key rotation.

Summary of commits in this PR:
- commit 1: scaffold tenant-subnet-allocator module, basic variables & outputs
- commit 2: add allocator core: lease resource, address arithmetic helpers, and acceptance tests
- commit 3: gpu-nodeclass-orchestrator module and example usage in private-installer chart
- commit 4: iam-shard templates and rotation helper script
- commit 5: state-prefix-migrator CLI + docs and migration plan templates
- commit 6: integration tests, CI pipeline updates, lint fixes

Changed areas / notable diffs:
- terraform/modules/tenant-subnet-allocator/main.tf: primary new logic (lease resource + allocation math).
- terraform/modules/gpu-nodeclass-orchestrator/*: new module coordinating nodeclass lifecycle and tenant binding CRs.
- terraform/modules/state-prefix-migrator/migrate.sh: posix script to perform safe state copy and tombstone writes.
- helm/charts/private-installer/values.yaml: new 'allocator' and 'nodeclass' sections and rollout knobs.
- installer/upgrade-hooks: plan-fence.sh was added to enforce sequential plans for overlapping tenant ranges.
- tests/integration: new concurrency tests flagged as flaky in CI until resource limits were adjusted.

Review conversation (high level):
- Maya Chen (reviewer): asked for clearer invariants on lease immutability and requested that the migrator perform a checksum validation post-copy. Response: added a post-copy checksum step and documented the failure modes.
- Omar Reyes (reviewer): raised concern about optimistic locking in TFstate leading to possible orphaned leases. Response: added tombstone markers and an idempotent reclaim path in migrate.sh and the plan-fence hook to mitigate overlapping applies.
- Priya Patel (reviewer): requested IAM role scoping examples for customers with STS federation. Response: added examples and a rotation helper that integrates with KMS.
- Threaded comments: small follow-ups included renaming variables for clarity and reducing the default pool size in CI to avoid flaky test failures. All requested code changes were addressed in follow-up commits.

CI and validation:
- Initial CI run: lint and unit tests failed due to shellcheck errors in migrate.sh and a missing test fixture. Fixed in commits 5 and 6.
- Rerun CI: all checks passed (unit, integration smoke, terraform validate). The integration pipeline includes a simulated BGP pool exhaustion scenario.

Migration and rollout plan:
1) Ship modules behind a feature flag in helm values; no behavior change for existing deployments by default.
2) For customers opting in, run state-prefix-migrator in dry-run mode and validate checksums.
3) Apply tenant-subnet-allocator for the tenant while plan-fence is enabled to ensure no concurrent claims.
4) Promote GPU nodeclass binding by updating tenant binding manifests; orchestrator will reconcile node affinity without destroying existing pools.
5) Once validated, flip the feature flag and remove the migration guard after a scheduled maintenance window.

Risk & rollback:
- Risk: partial migration leaving tombstone markers that would block reclaims. Mitigation: migrator can revert tombstones and plan-fence prevents concurrent operations during migration window.
- Rollback: rollback/rollback-checker.go detects partially-applied migrations and prevents a destructive apply until an operator clears the migration marker or runs the migrator revert path.

Why this is different from previous work: Existing PRs introduced TFstate sharding and lease concepts but were focused on subnet tagging and GPU pool labeling. This change couples a BGP-aware allocator with explicit tenant-prefixed TFstate migration tooling and a dedicated GPU nodeclass orchestrator to reduce churn and simplify tenant onboarding semantics.

Merge outcome: After addressing reviewer feedback and fixing CI shellcheck/test fixture failures, the branch was squashed and merged to main on 2025-11-10. Release notes were prepared and a rollout runbook was added to docs/private-deployments/tenant-networking.md.

Follow-ups (tracked):
- ENG-3478: Add observability hooks (metrics & traces) to allocator for allocation latency and failures.
- ENG-3499: Expand integration tests to cover cross-region allocations for multi-AZ private installs.
- PM-128: Evaluate UX knobs for self-serve tenant onboarding using the allocator API.
Adds tenant-scoped BGP subnet allocator TF module, GPU nodeclass orchestrator module, tenant-scoped IAM sharding templates, and a state-prefix migrator to enable per-tenant TFstate layout. Includes upgrade/rollback fences and integration tests. No breaking changes expected for existing deployments.
