# Private installer: provision cross-account export sinks + rollbackable upgrade script for audit-log-exporter

Context and motivation:
This PR wires audit-log-exporter into the Private installer/Terraform stack so customers deploying in a VPC/on-prem flow can opt into automated export sink provisioning (S3/GCS) and have an auditable, rollback-capable upgrade for existing deployments. Customers frequently asked for a single, supported path to provision export destinations with the proper retention and KMS settings; manual steps were error-prone and caused non-compliant buckets during audits (see linked SUP-8810).

What this change does (high level):
- Adds terraform/modules/audit-sink: a modular Terraform module that supports aws_s3 (cross-account) and gcp_storage buckets, optional customer-managed KMS keys, and an iam role/binding for the audit exporter service account. Module includes idempotent retention enforcement and an optional lifecycle rule template.
- Extends charts/private-installer: adds templates/audit-sink.yaml and values.auditSink to enable installer-driven sink wiring and to pass sink credentials/secrets into the audit-log-exporter Helm release via sealed secrets when using Private (VPC) installs.
- Installer hooks and scripts: adds scripts/installer/upgrade-audit-sink.sh which performs a dry-run, snapshots current bucket policies/policy-versions, applies module changes, and produces a rollback manifest that can be applied to revert IAM/policy changes if something unexpected happens during install.
- Adds reconciler job: jobs/reconcile-sinks/k8s-job.yaml and a small controller shim used only in the installer to reconcile retention settings across existing sinks for customers upgrading from older installer versions.
- E2E test harness (CI-only): test/e2e/audit-sink-provisioning includes a lightweight cloud-provider shim that simulates sink creation and verifies retention/policy enforcement and that the audit exporter can authenticate to the provisioned sink.

Commits (summary):
- c4f9a7d  Priya Raman  Add terraform module for audit sink (aws/gcs) + docs
- a1b2c33  Priya Raman  Helm chart: add audit-sink template and values schema
- b5d2e1f  Priya Raman  Installer: add upgrade-audit-sink.sh + rollback snapshot tool
- f6a7d88  Noah Chen     Reconciler job + e2e test harness (CI only)
- 9d3c5af  Meera Patel   Fix linter: helm template indentation and terraform fmt

Files/areas changed (high level):
- terraform/modules/audit-sink/main.tf (new): implements provider-agnostic resources, KMS optional block, iam role attachment
- terraform/modules/audit-sink/outputs.tf (new)
- charts/private-installer/values.yaml (modified): added values.auditSink.{enabled,type,retentionDays,kmsKey}
- charts/private-installer/templates/audit-sink.yaml (new): Helm template to generate sealed secret and wiring
- scripts/installer/upgrade-audit-sink.sh (new): upgrade/rollback orchestration script with --dry-run flag and --snapshot-path output
- jobs/reconcile-sinks/k8s-job.yaml (new)
- test/e2e/audit-sink-provisioning/* (new test harness)
- docs/private/audit-sinks.md (new): operator-facing docs and troubleshooting steps

Small illustrative snippet (values):
values.auditSink: |
  enabled: true
  type: aws_s3 # aws_s3 | gcs
  retentionDays: 3650
  kmsKey: optional-customer-kms-arn

Review conversation (abridged):
- Meera Patel (comment, 2025-11-10): "This is a big change for Private installs. Please add an explicit migration plan for customers that already have an S3 bucket with weaker retention. We should not change retention silently."
  - Priya Raman (reply, 2025-11-10): "Agree. added the reconciler job that will surface non-compliant buckets as part of the installer preflight and will require an explicit --enforce-retention flag to mutate retention settings. Documented in docs/private/audit-sinks.md."
- Noah Chen (request changes, 2025-11-12): "Terraform module should expose a kms_key_id output and avoid creating keys by default. Also update the Helm chart to accept an existing sealed secret name so customers can inject pre-provisioned credentials."
  - Priya Raman (reply, 2025-11-13): "Addressed: module now has kms_key_create boolean; outputs.kms_key_id; Helm values accept .existingSealedSecretName. Added test case in e2e harness."
- Diego Alvarez (approval, 2025-11-18): "Verified reconciler job and installer dry-run; e2e simulation shows correct wiring. Approve."

CI and gating: initial CI run (2025-11-14) failed helm-lint due to template indentation; fixed in 9d3c5af and terraform fmt adjusted. Final CI run (2025-11-19) shows: build:pass, unit-tests:pass, helm-lint:pass, terraform-fmt:pass, e2e-sim:pass. ci_status: pass.

Rollout and upgrade notes for operators (from PR):
- Default behavior: audit sink provisioning is opt-in (values.auditSink.enabled=false). When enabled, the installer will create resources using the audit-sink Terraform module.
- Upgrading existing installs: run scripts/installer/upgrade-audit-sink.sh --dry-run which emits a snapshot and a proposed plan. To apply changes that mutate retention or IAM, pass --enforce-retention to allow the reconciler to update bucket lifecycle and retention settings. The script produces a rollback manifest under the provided snapshot-path that can be used to revert IAM/bucket policy changes.
- Rollback guarantees: we snapshot IAM roles/policy-versions and bucket lifecycle rules prior to any changes. The rollback manifest will re-apply older versions and rotate any temporary credentials created by the installer.

Security/compliance considerations:
- The module supports customer-managed KMS, cross-account roles, and grants with least-privilege scoped to upload/putObject and list for the audit exporter principal.
- Audit events describing the provisioning actions are emitted to a local compliance log under /var/log/redwood/installer-audit.log in the host running the installer, and are retained according to the operator-configured retention window.

Why this approach?
Manual provisioning caused repeated incidents where retention or KMS settings were misconfigured. Centralizing the provisioning in a tested Terraform module and wiring it through the Private installer reduces manual steps, provides an auditable change history, and gives customers a safe rollback path for upgrades.

Follow-ups (future work noted in PR):
- Add Azure Blob support to terraform/modules/audit-sink (engineering spike scheduled ENG-7390).
- Improve the reconciler to run as a long-lived k8s controller in the next major release so that drift is continuously corrected if operators opt-in.

Merge outcome:
- PR merged via squash on 2025-11-20 by Priya Raman (merge_commit c4f9a7d).
- Post-merge: chase-up docs update PR queued against redwood-quickstarts to include a Private quickstart for audit sink provisioning.

If you want to try the flow: see docs/private/audit-sinks.md for step-by-step and scripts/installer/upgrade-audit-sink.sh --dry-run to preview changes.
Automated provisioning and safe upgrade path for private audit export sinks: 
- New Terraform module to create cross-account S3/GCS sinks with IAM/KMS options.
- Helm chart values and installer hooks to wire audit-log-exporter to provisioned sinks.
- Safe upgrade script that snapshots policies and provides a rollback path.
