# Introduce subnet quarantine controller and automated route failback

Motivation: private control planes occasionally observe transient subnet-level egress failures (malformed route table update, NAT gateway flapping, or AZ-level entitlement issues). Previously this required manual intervention to remove affected subnets from control-plane routing and reattach them after remediation. This PR introduces an automated, reversible quarantine mechanism plus a route failback operator to ensure control plane availability during transient egress incidents without human-in-the-loop recovery.

Summary: implement a Kubernetes controller (subnet-quarantine-controller) that watches health probes and Terraform state annotations to mark subnets as 'quarantined' when egress or endpoint checks fail for a configurable threshold. When quarantined, the operator updates route-table attachments (via Terraform-managed route resources with a short-lived conditional override) to fail traffic to a preconfigured egress fallback (secondary NAT/proxy or regional egress VPC). It also flips Route53 conditional forwarding entries or Private Hosted Zone records in a controlled, rate-limited way to maintain DNS reachability for control-plane endpoints. All changes are reversible and instrumented.

Design notes: the controller is implemented as an opt-in component deployed from the existing private-installer Helm chart. It communicates with Terraform-managed resources through a small idempotent helper that enqueues a Terraform run via our pipeline API (safe-plan-and-apply mode) rather than directly mutating cloud resources. The operator records decisions in annotations on Terraform-managed subnet resources and emits Audit events so the installer's reconciliation loop can surface the state. Quarantine TTL, probe thresholds, and rate limits are configurable via Helm values and a ConfigMap.

Compatibility: the change is backward-compatible when disabled (Helm value quarantine.enabled=false). For existing installations we provide a zero-downtime upgrade path: the operator is deployed in canary mode by default (enabled in staging and optional for on-prem via helm values). We added a migration script that adds the required subnet annotation schema so older Terraform states are not accidentally touched.

Testing and verification: unit tests for controller logic and probe thresholds; integration harness that simulates route table failures and verifies that route failback is applied and then restored when health returns; e2e run in staging that exercised cross-AZ failover; manual canary exercise performed with SRE team to validate rollback. See tests/e2e/subnet-failback-simulation.yaml for the harness.

Operational runbook: included docs/operational/runbook/subnet-quarantine.md describing alert thresholds, mitigation steps, and how to force-unquarantine a subnet. We also create Linear follow-ups for alert tuning and cost impact analysis.

Security and audit: changes to route tables and Route53 entries are recorded in audit logs and require a signed plan approval for production changes. The operator does not hold cloud credentials; it requests a signed apply token from the control plane which is limited-scoped and short-lived.

Why not direct cloud mutation: using the Terraform run API maintains single-source-of-truth for infrastructure state while allowing automated, auditable short-term overrides. This avoids drift between operator decisions and declarative Terraform state.

Priya Nair (2024-11-09): Can we avoid direct route table mutation? Please ensure all changes go through Terraform CI. - addressed; controller now triggers signed terraform run (Ethan)
Owen Zhang (2024-11-12): Add metrics for quarantine_count and failback_latency; gate default values behind Helm toggles. - addressed; added Prometheus metrics and helm values (Ethan)
Luis Martinez (2024-11-15): Concern about race between autoscaler and quarantined subnets. Suggest adding annotation so other controllers skip scaling for a quarantined subnet. - added skip annotation and documented behavior (Ethan)
Priya Nair (2024-11-20): CI initially failed on terraform fmt in module; reran formatter and CI passed. Approving now. - approval given
Maintainer merge comment (2024-11-22): Squashed and merged after staging canary passed; follow-up created ENG-3190 for alert tuning.
Merged via squash on main by maintainer on 2024-11-22 after approvals. Post-merge pipeline executed and deployed controller to staging canary; production rollout gated behind feature flag and manual approval. Follow-ups: ENG-3190 (alert tuning), SRE-923 (cost analysis of fallback egress).
Add optional subnet quarantine controller and automated route failback. Controlled via helm value 'quarantine.enabled'. Adds new Terraform module 'route-failback' and migration script for subnet annotations. No breaking changes when feature is disabled. See docs/operational/runbook/subnet-quarantine.md for operational details.
