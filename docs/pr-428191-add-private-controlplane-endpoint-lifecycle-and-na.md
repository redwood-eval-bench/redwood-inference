# Add private control-plane endpoint lifecycle manager and namespace DNS mirroring

Summary: This PR introduces a control-plane endpoint lifecycle manager for the private installer along with an optional namespace-level DNS mirroring mode that keeps customer-private control-plane DNS names resolvable across shared and delegated zones.

Motivation: Customers deploying Redwood Private in VPCs across accounts and AZs ran into two recurring issues: (1) transient endpoint drift during rolling upgrades / node replacement that left stale ENIs and SG attachments; (2) split-horizon DNS complexity for cross-account privatelink setups where control-plane services live in different route domains. The goal is to make endpoint lifecycle deterministic, idempotent, and auditable, and to provide a low-risk DNS mirroring option to avoid manual route53 delegation edits.

Design:
- endpoint-lifecycle-manager: a reconciler (installer controller) that performs create/refresh/revoke cycles for control-plane privatelink endpoints. It tracks resource intent via tags and a small controlplane-state SSM parameter for cross-run coordination. The reconciler enforces graceful drain: a handshake that verifies active connections and KV cache warmup before swap.
- ephemeral-auth-tokens: short-lived IAM policies attached during endpoint creation to allow cross-account accept/attach without giving permanent rights. Tokens are rotated on every successful reconciliation.
- namespace DNS mirroring: an opt-in installer flag (values.controlplane.dnsMirrorNamespaces) that causes the installer to maintain CNAME records in a mirror zone (customer-controlled) and create conditional forwarding entries in the control-plane VPC resolver. Mirror entries are updated in a staged fashion to avoid TTL cliffs.
- Rollout semantics: canary by namespace; can be executed per-region. Reconciler supports a dry-run mode and a staged commit strategy that lists planned changes before applying them.

Implementation highlights:
- Terraform: new module modules/controlplane/endpoint_lifecycle with idempotent create/update hooks, prereq checks for IAM roles and route53 permissions. Added outputs for mirror zone name and change-batch id.
- Helm: charts/controlplane/templates/endpoint-reconciler-deployment.yaml and values.yaml flags for dnsMirrorNamespaces, reconcilerConcurrency, and dryRun.
- Installer: pkg/installer/reconciler/endpoint_manager.go implements the state machine (intent -> provision -> attach -> verify -> publish-dns -> finalize) and exposes metrics for reconciliation duration and transient failures.
- Scripts: scripts/upgrade/rollback-ephemeral-endpoints.sh to assist migration during emergency rollbacks.
- Tests: tests/e2e/controlplane/dns_mirroring_test.go added to validate TTL-safe updates, and tests/integration/endpoint_reconcile_test uses a mocked AWS API with recorded fixtures.

Rollout plan:
1) Merge to main behind feature-flag disabled by default.
2) Enable reconciler in a staging account with dnsMirrorNamespaces set for a single namespace and run dry-run for 24h.
3) Flip to commit mode and monitor reconciliation metrics & Route53 change-batches.
4) Canary to one production account/region for 48h.
5) Full enable across accounts.

Rollback: reconciler supports revert-to-previous-state which re-applies last-change-batch for DNS and reattaches previous ENI for 10m window; scripts/upgrade/rollback-ephemeral-endpoints.sh provides an operator workflow.

Observability:
- New metrics: controlplane_endpoint_reconcile_duration_ms, controlplane_endpoint_reconcile_errors_total, controlplane_dns_mirror_change_batches_total.
- Audit logs: reconciler writes audit entries to a reconciler-audit S3 prefix with JSON debug for every decision to aid postmortems.

Migration notes:
- Operators must grant route53:ChangeResourceRecordSets and route53:ListHostedZones to the installer role if dns mirroring is enabled.
- Cross-account acceptance still requires delegated IAM trust; ephemeral tokens avoid long-lived roles but require sts:AssumeRoleWithWebIdentity in some setups.

Testing and validation:
- Unit tests for the state machine with simulated transient AWS errors.
- Integration tests with localstack recording of route53 and ec2.
- CI: terraform-validate, helm-lint, unit, integration.

Security review: scoped ephemeral tokens reviewed by security; recorded in SUP-9812. No plaintext secrets introduced.

Compatibility: The feature is opt-in and behind a Helm flag; existing installs are unaffected.

Files changed (high level): modules/controlplane/endpoint_lifecycle/main.tf, modules/controlplane/iam_roles.tf, charts/controlplane/values.yaml, charts/controlplane/templates/endpoint-reconciler-deployment.yaml, pkg/installer/reconciler/endpoint_manager.go, pkg/installer/reconciler/state_machine_test.go, scripts/upgrade/rollback-ephemeral-endpoints.sh, tests/e2e/controlplane/dns_mirroring_test.go, docs/privatelink-dns-mirroring.md, examples/controlplane/dns-mirror-example.yaml, ci/terraform-validate.yml

Commits: - "init: add endpoint lifecycle terraform module and basic plan" - "feat(reconciler): implement endpoint lifecycle state machine" - "chore(helm): add reconciler deployment + values" - "test: add e2e dns mirroring tests" - "docs: add privatelink DNS mirroring operator guide" - "ci: add terraform and helm validation steps"

Review conversation: ["Marco Li: Overall this looks good; can we make the dry-run output machine-readable (JSON) so automation can detect planned changes?","Priya Menon: Added a --output=json option and included it in the reconciler CLI; updated docs.","Ana Rodriguez: We need explicit retries for Route53 throttling errors; add exponential backoff with jitter.","Priya Menon: Implemented retry wrapper and added recorded integration tests for 429 handling.","Samir Patel: Confirmed no long-lived IAM policies created; can you add an audit event for token issuance?","Priya Menon: Added reconciler-audit S3 write for every ephemeral token issuance and rotation.","Ana Rodriguez: Please add a brief runbook for operator rollback.","Priya Menon: Included rollback script and runbook section in docs/privatelink-dns-mirroring.md"]

CI checks summary: ["unit: pass (34s)", "lint: pass (12s)", "terraform-validate: pass", "helm-lint: pass", "integration: pass (retries: 1)", "e2e: pass (flaky: rerun succeeded)"]

Changed files (detailed list): ["modules/controlplane/endpoint_lifecycle/main.tf", "modules/controlplane/endpoint_lifecycle/outputs.tf", "modules/controlplane/iam_roles.tf", "charts/controlplane/values.yaml", "charts/controlplane/templates/endpoint-reconciler-deployment.yaml", "pkg/installer/reconciler/endpoint_manager.go", "pkg/installer/reconciler/state_machine_test.go", "scripts/upgrade/rollback-ephemeral-endpoints.sh", "tests/e2e/controlplane/dns_mirroring_test.go", "docs/privatelink-dns-mirroring.md", "examples/controlplane/dns-mirror-example.yaml", "ci/terraform-validate.yml"]

Notes on idempotency and safety:
- The Terraform module is written with lifecycle ignore_changes on ephemeral metadata tags to avoid churn when reconciler rotates tokens.
- Reconciler will not perform destructive detach operations unless the verify step confirms no active connections OR the operator explicitly sets force=true.
- DNS mirror changes are applied in a two-phase commit: prepare (create new entries with suffix .staging), verify resolution, swap CNAMEs, then garbage collect staging entries.

Merge outcome: merged to main via squash on 2025-11-12 after approvals from Marco, Ana, and Samir.

Next steps: monitor new metrics for 2 weeks, collect feedback from on-call, and prepare a short postmortem template to capture any reconciliation incidents.
Introduce endpoint lifecycle reconciler for private control plane and enable optional per-namespace DNS mirroring for cross-account Privatelink setups. No user-facing API changes; operator-facing config knobs added.
Marco Li: Overall this looks good; can we make the dry-run output machine-readable (JSON) so automation can detect planned changes?
Priya Menon: Added a --output=json option and included it in the reconciler CLI; updated docs.
Ana Rodriguez: We need explicit retries for Route53 throttling errors; add exponential backoff with jitter.
Priya Menon: Implemented retry wrapper and added recorded integration tests for 429 handling.
Samir Patel: Confirmed no long-lived IAM policies created; can you add an audit event for token issuance?
Priya Menon: Added reconciler-audit S3 write for every ephemeral token issuance and rotation.
Ana Rodriguez: Please add a brief runbook for operator rollback.
Priya Menon: Included rollback script and runbook section in docs/privatelink-dns-mirroring.md
