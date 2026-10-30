# Converge sealed-secrets and Vault workflows into kms-binding-operator and add rotation-scheduler

Context: Private installs have two parallel secret management paths: sealed-secrets for k8s-native secrets and a set of bespoke Vault/KMS scripts used by the installer. This divergence makes automated TLS/KMS rotation and safe rollbacks brittle across upgrade/rollback cycles.
Motivation: unify runtime secret consumption and rotation into a single operator (kms-binding-operator) that speaks to HSM-backed KMS providers, supports envelope encryption, and exposes a controlled rotation lifecycle. This PR introduces a Helm subchart (rotation-scheduler) and a small Terraform module to manage KMS aliases used by envelope keys.
Goals: 1) provide a canonical binding path for workloads to fetch decrypted secrets via k8s CSI-like mounts, 2) centralize rotation orchestration with staged canaries and automated rollback hooks, 3) remove ad-hoc sealed-secrets reconcilers in favor of an operator that can emit sealed-backups for offline recovery.
What changed:
- New Helm chart: charts/rotation-scheduler (CronJob-based scheduler + leader-election + webhook preflight).
- New operator: deploy/kms-binding-operator manifests + RBAC, CRDs (Binding, EnvelopeKey, RotationPlan).
- Terraform: modules/kms/alias (creates KMS alias, IAM roles, and outputs alias arn).
- Installer hooks: installer/helm/templates/preupgrade-checks.yaml and installer/hooks/staged-rollback-hook.yaml to ensure idempotent upgrades and safe rollback windows.
- Migration tool: bin/migrate-sealed-to-kmsbinding.sh (idempotent, dry-run capable)
- Docs and quickstart: docs/private/kms-binding-migration.md
Rollout strategy included in PR: 1) install rotation-scheduler in staging with rotationPlan: dry-run mode, 2) deploy kms-binding-operator with canaryBinding for non-critical namespaces, 3) enable envelope-proxy on a subset of nodes to validate HSM KMS latency under load, 4) schedule low-impact TLS cert rotation using a 72-hour staged window, 5) monitor metrics (rotate_success_rate, rollback_triggered, envelope_latency_p50/p95) and only widen rollout on success.
Upgrade and rollback notes:
- The preupgrade-checks hook validates that every sealed-secret has a corresponding Binding CR; it will fail fast if there are secrets without migration mapping.
- The staged-rollback-hook keeps a stable copy of the previous EnvelopeKey and automatically rebinds workloads if the rotation fails within the configured grace period.
- The Terraform module supports a "promote_allowlist" variable so aliases can be created ahead of time in a safe, auditable step.
Security & audit: operator emits audit events to the existing installer audit sink and supports optional KMS key wrapping via customer HSM. Migration utility runs with dry-run by default; operator requires an explicit enablement flag in values.yaml.
Testing performed: e2e in benchmark-lab (integration with simulated HSM), unit tests for controller reconciliation, manual runbook walkthrough.
Checklist:
- [x] Helm subchart implemented
- [x] Operator manifests and CRDs
- [x] Terraform alias module
- [x] Migration script + dry-run
- [x] Pre-upgrade and rollback hooks
- [x] Docs and runbook
- [x] CI pipeline passing
Known limitations: multi-cluster sealed-secret federation is out-of-scope for this change and will be handled in a follow-up (ENG-1829).
Samir Gupta: Overall direction LGTM. Two asks: (1) add an ownerReference to EnvelopeKey so sealed-backups are garbage collected, (2) ensure the Terraform module emits a non-sensitive plan output for audit.
Avery Chen (author): Addressed both; added ownerReference and outputs.tf with alias_arn and created_by tags. Also added tests for plan output.
Leah Thompson: Please add a migration safety-net: if migration encounters unmapped secrets, operator should create a migration-ticket CRD instead of blocking the whole install. Also add metrics for migration_progress.
Avery Chen: Added MigrationTicket CRD and migration_progress metric. Updated docs with runbook step to manually triage tickets.
Diego Morales: CI flaky on e2e (simulated HSM) — add retry to the integration harness and a job-specific timeout. Approving once green.
Avery Chen: Increased timeout and added isolated e2e retry. Rerunning CI.
- Introduced kms-binding-operator to centralize secret consumption and rotation.
- Added rotation-scheduler Helm subchart to orchestrate staged TLS/KMS rotations.
- Terraform module for KMS alias creation and IAM role scaffolding.
- Migration tool to move sealed-secrets to the operator-managed binding model.
- Installer hooks for preupgrade checks and staged rollback coordination.
