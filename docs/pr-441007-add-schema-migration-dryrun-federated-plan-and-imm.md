# add-schema-migration-dryrun-federated-plan-and-immutable-migration-records

Problem: Private deployments run schema migrations during upgrades and customers have reported hard-to-reproduce downtime when migrations make irreversible changes. Existing tooling supports full migrations and snapshot-based rollbacks, but we lack a lightweight, repeatable dry-run and audit trail that operators can run against an isolated tenant snapshot or a simulated topology.

This PR introduces a migration dry-run harness and a federated migration plan that can be executed without touching live write paths, plus immutable migration-record objects for audit and safe rollback planning. Key goals: 1) let operators validate migration effects (row counts, FK integrity, index usage) against exported snapshots, 2) run plans in parallel across federated components (operator, control-plane DB, tenant DBs) and aggregate results, 3) provide an immutable record of what a migration DID or WOULD do so rollbacks and postmortems have a single source of truth.

What this change does (high level):
- Adds a migration-simulator sidecar container image and Kubernetes Job template (k8s/migration-dryrun-job.yaml) that runs migrations against a read-only, mounted snapshot volume or a Postgres logical replica. The simulator runs in a no-op mode that records SQL plan effects instead of applying destructive DDL when `DRY_RUN=true`.
- Introduces a new CRD `MigrationRecord` (api/v1alpha1/migrationrecord.yaml) that is immutable after creation. The operator writes a MigrationRecord for every dry-run and every real migration; records include a checksum of the migration artifact, the computed diff summary, and an operator-signed attestation blob.
- Adds a federated planner implementation (pkg/planner/federated_plan.go) that composes per-tenant and control-plane steps into an ordered execution graph, allowing dry-run execution to run in parallel where safe and to fail fast on schema drift. The planner emits a deterministic plan ID used to tie MigrationRecords together.
- Extends the installer CLI (`ri-installer migrate --dry-run`) to produce an exportable JSON report (artifacts/migration-reports/<plan-id>.json) and to optionally push the MigrationRecord into the cluster. The CLI includes a `--terraform-check` mode that runs `terraform plan` against generated infra diffs to detect resource drift that migrations might rely on.
Diego Alvarez: Nice separation of concerns; can we ensure MigrationRecord attestation is KMS-backed? Suggest adding optional KMS keyref in values.yaml.
Maya Chen (author): Added optional kmsKeyRef to helm values and the record controller now signs with the cluster KMS when present.
Priya Raman: Planner looks solid but add a stronger check for FK cascade semantics in the dry-run so we don't underestimate row-level effects.
Maya Chen (author): Added FK cascade check and an integration test that runs a simulated cascade on a 3-table fixture.
Ellen Park: CI flake on test TestPlannerParallelBranches; rerun fixed it, all green. Approving after tests pass.
Added migration dry-run harness, federated planning for upgrades, immutable MigrationRecord CRD, helm pre-upgrade dry-run hook, and optional terraform drift-check from the installer CLI.
