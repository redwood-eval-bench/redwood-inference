# Automate TLS cert rotation via SealedSecrets + Vault + KMS; Helm & Terraform changes

Motivation: Private deployments require reproducible, auditable TLS certificate rotation with least-privilege key usage. Operators currently perform manual cert updates using ad-hoc scripts; this creates downtime risk and audit gaps. This PR implements an automated rotation pipeline that ties together Kubernetes SealedSecrets, HashiCorp Vault (transit & PKI), and cloud KMS envelope keys, plus Terraform modules and Helm hooks to orchestrate upgrades and safe rollbacks.

What this PR does (high level):
- Adds a new Terraform submodule `secret-rotation` to provision KMS keys, Vault policies, and an auxiliary state bucket/lock for rotation metadata.
- Adds a Helm chart patch and lifecycle hooks for the installer that bootstrap sealed secrets and a rotation controller on upgrade-install.
- Introduces a rotation controller (lightweight Go binary) that: obtains new certs from Vault PKI, envelopes private keys with cloud KMS, seals resulting Kubernetes Secret using SealedSecrets controller, applies to cluster, and monitors readiness before switching ingresses.
- Migration tool: a one-shot job to convert existing unsealed TLS Secrets into SealedSecrets while recording previous keys for safe rollback.
- E2E harness updates: rotation acceptance test and canary rollout scenario in the perf-canary suite.

Checklist: 
- [x] Terraform plan + apply tested against staging account (uses mock KMS in CI)
- [x] Helm chart dry-run and post-install hook tested in KIND cluster
- [x] Migration job tested against staging clusters with old certs
- [x] RBAC tightened for controller (Minimal roles for writing SealedSecret and reading KMS/Vault)
- [x] Audit logs wire-up: Vault audit + KMS key use events documented

Design notes: 
- SealedSecrets are used as the canonical persisted resource; the rotation controller seals new keys and never stores unencrypted private keys in cluster state. 
- Vault PKI is the CA; the controller requests issuance and relies on Vault TTL/renewal endpoints. Private key material is envelope-encrypted with cloud KMS (envelope keys are managed by Terraform-created KMS keys). 
- Terraform module creates: KMS key (with rotation enabled), Vault policy for the controller, and a small S3/GCS bucket to hold rotation metadata (last-rotated, previous-key-id) with object-level ACLs.
- Helm hooks: pre-upgrade installs the migration job (if needed) and post-upgrade triggers the controller canary rollout for cert swap. Hooks are idempotent and have deadlines to avoid blocking installs.

Rollback & safety: 
- Every rotation stores the previous sealed secret and KMS ciphertext id in rotation metadata. The controller exposes an endpoint to initiate a rollback which re-applies the previous SealedSecret and swaps ingress TLS atomically.
- Canary flow: controller rotates a single ingress (or a percentage via weighted ingress) first, waits for health checks, then proceeds. Failures auto-revert using stored metadata.

Testing: 
- Added unit tests for envelope encryption and SealedSecrets pack/unpack. 
- New e2e test exercises full path: issue cert -> controller seals -> apply -> readiness -> rotate back.

Related: Linear ENG-3145, SRE-212; Support ticket SUP-1122 for customer migration request.

Notes on deployment: Run `terraform apply` for `secret-rotation` first, then upgrade installer chart with `--set rotation.enabled=true`. See notes_for_ops field for step-by-step.

This description contains the full set of changes, migration steps, and rollback procedures.

Miguel Ortiz: Requested change - ensure previous key material is retained in an immutable store and not garbage-collected immediately; add TTL-based GC with operator override.
Priya Nair: Reply - added rotation metadata bucket and GC policy in Terraform; operator override endpoint included in controller (see controller/ops.go).
Emily Chen: Comment - suggestion to split KMS provisioning into separate apply step for infra-only customers; implemented via optional module flag and documented deploy ordering.
Samir Patel: Approve with comment - please add one more e2e for cross-region KMS latency; filed follow-up ENG-3210.
Emily Chen: Approval - verified RBAC and Vault policy least-privilege in staging cluster; ready to merge.
Automated TLS certificate rotation for private installs: introduces a rotation controller that integrates Vault PKI, cloud KMS envelope encryption, SealedSecrets, Terraform modules for infra, and Helm hooks for safe upgrades. Includes migration tooling and rollback safety. Non-breaking; opt-in via rotation.enabled=true.
1) Apply terraform: cd terraform/modules/secret-rotation && terraform init && terraform apply -auto-approve
2) Upgrade installer Helm chart with: helm upgrade --install redwood-installer ./helm/installer -f values.yaml --set rotation.enabled=true
3) Monitor canary: controller exposes /health and /rotation-status; wait for canary success before full rollout.
4) To rollback a failed rotation: curl -XPOST -H 'Authorization: Bearer <token>' https://controller.example.internal/rollback?ingress=<name> -- the controller will re-apply stored sealed secret and swap ingress certs.
5) Ops should confirm KMS key policies and Vault audit are present; see docs/ops/rotation-runbook.md for troubleshooting.
