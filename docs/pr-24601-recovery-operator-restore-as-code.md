# introduce-recovery-operator-and-restore-as-code-playbooks

Motivation: customers running Redwood Private in regulated environments asked for a repeatable, testable way to orchestrate multi-AZ failovers and post-failover restores as code. Previously we had ad-hoc runbooks and manual Helm/Terraform steps; this PR introduces an opinionated Recovery Operator (K8s controller) plus a set of versioned "restore-as-code" playbooks that automate DR drills and restores for on-prem/VPC customers.

What this PR does: 1) Adds a lightweight Kubernetes operator (cmd/recovery-operator) that watches RecoveryPlan CRDs and coordinates: snapshot import, PVC remapping, configmap migrations, and model cache warmup. 2) Introduces a new helm chart subchart recovery-operator/ with configurable RBAC, leader-election, and dry-run mode. 3) Adds structured restore playbooks under installer/playbooks/restore-as-code/ (YAML templates driven by the operator) covering common scenarios: AZ evacuation, primary datastore failover, and full cluster restore from object-store snapshots. 4) Terraform module updates: installer/terraform/modules/vpc-recovery/ to expose S3/MinIO snapshot IAM roles and lifecycle policies; module includes test bucket lifecycle for drills. 5) Upgrades upgrade/rollback tooling: installer/upgrade/rollback.sh can now generate a RecoveryPlan manifest and submit it to the operator; the script validates preconditions and can perform automatic rollback if restore verification fails. 6) Adds test harness and CI flows: chaos-run e2e job that exercises simulated AZ failover via taints + snapshot/restore path.

Usage summary: - To run a drill: ./installer/playbooks/restore-as-code/run_drill.sh --plan az-evacuation --namespace redwood --dry-run=false; this will create a RecoveryPlan CRD that the operator executes. - To perform an emergency restore: installer/upgrade/rollback.sh --restore-from s3://redwood-dumps/2024-10-30 --verify true.

Backward compatibility & safe defaults: Operator is opt-in via helm values: recoveryOperator.enabled=false by default. Playbooks are additive; Terraform module exposes new output variables but preserves existing API. RBAC and KMS integrations are gated by values and require explicit enabling for customer environments.

Testing & validation: - Unit tests for operator reconciliation and plan state machine. - Integration terratest that provisions a small cluster and runs a restore-as-code drill against MinIO-backed snapshots. - CI added a chaos-run job that flips AZ labels and checks that the operator triggers plan execution and that the verification hooks pass (synthetic traffic + KV cache checksum).

Rollout plan: feature gated behind values; initial rollout to internal staging clusters for two weeks, followed by opt-in beta for customers on Private. Addendum: will publish runbook docs to docs/private/recovery-operator.md and a step-by-step video for SREs.

Related: ENG-3421 for operator implementation details; SUP-5698 customer escalation that prompted automation; SRE-1186 follow-up to add long-term alerting for plan regressions.
Maya Ramirez: Nice end-to-end approach. Two questions: (1) Can the operator be run in a read-only "simulate" mode so customers can validate without touching PVCs? (2) The Terraform changes add an IAM role — did we lock down permissions to only list/get/put objects needed for restore?
Owen Patel: Responding: yes, operator supports dry-run (values.recoveryOperator.dryRun=true) and we added an explicit simulate command in run_drill.sh; terraform role scope is restricted to s3:GetObject, s3:PutObject, s3:ListBucket and kms:Decrypt for the snapshot key. Added policy snippet to module docs.
Priya Singh: Minor nit: reconciliation backoff needs caps to avoid thundering herd; please add jitter and a max backoff of 5m.
Owen Patel: Added jitter + capped backoff to reconciler and updated unit tests to cover backoff behavior.
Ethan Zhao: Integration tests flaky on CI once; re-ran job and it passed. Consider adding an annotation to skip heavy restore tests on PRs that touch unrelated code.
Owen Patel: Added ci annotation skip tag and limited terratest cluster size; also split chaos-run into nightly job to reduce PR noise.
Maya Ramirez: Approving after changes. Marking as approved for merge.
Add opt-in Recovery Operator and restore-as-code playbooks to automate DR drills and restores for Private deployments. Includes Helm chart, Terraform module updates, and upgrade/rollback tooling changes. Operator disabled by default; see docs/private/recovery-operator.md for usage and migration notes.
Operator is opt-in. To enable in a cluster: helm upgrade --install recovery-operator charts/recovery-operator --set recoveryOperator.enabled=true --namespace redwood. Run a dry-run drill first: ./installer/playbooks/restore-as-code/run_drill.sh --plan az-evacuation --dry-run=true. Monitor CI chaos-run job and alert on RecoveryPlanFailed metric in the SRE dashboard.
New: RecoveryPlan CRD, recovery-operator controller, restore-as-code playbooks; Terraform vpc-recovery module with snapshot IAM; upgrade script restore flow. Tests and docs included.
1) Deploy operator in staging with dryRun=true
2) Execute run_drill.sh for az-evacuation in staging
3) Verify verification hooks (synthetic traffic + cache checksum) return success
4) Toggle dryRun=false and execute full restore against MinIO
5) Validate rollback when verification intentionally fails
6) Review CI chaos-run logs for plan execution timing
