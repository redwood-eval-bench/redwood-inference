# Introduce ephemeral control-plane ENIs and zone-bound PrivateLink allocator

Motivation: For private deployments we need a safer control-plane networking model that avoids persistent, account-scoped ENIs and reduces blast radius when rolling control-plane upgrades. This PR introduces two complementary changes: (1) ephemeral control-plane network interfaces (ENIs) that are created/owned by the installer during control-plane pod lifecycle and torn down after handoff, and (2) a zone-bound PrivateLink allocator that pins PrivateLink endpoint interfaces to AZ-local subnets with soft-affinity and deterministic fallback.

What this PR changes: single continuous description of changes across codepaths: added a new Terraform submodule terraform/modules/controlplane-enis which implements aws_network_interface resources with lifecycle create_before_destroy + tags and an ownership marker so the installer operator can claim and drain them. The module exposes inputs for subnet_id, security_group_ids, and a new "ephemeral_lifetime" TTL used by the installer to garbage-collect stale ENIs. The PrivateLink allocator is implemented in installer/terraform/modules/privatelink-allocator and the control-plane Helm chart: it uses a small allocation table stored as a DynamoDB (or statefile for on-prem) keyed by vpc_id+service_name to grant zone-scoped prefix reservations and to emit route table associations to the controlplane route manager.

Installer/Helm changes: modified helm/redwood-private/templates/controlplane-svc.yaml to add a new sidecar (controlplane-net-allocator) and a ConfigMap template with allocation policy parameters (preferred_azs, sticky_ttl_seconds, failover_priority). Added hooks in helm charts for staged rollout: pre-upgrade job that migrates sticky PrivateLink assignments, and a post-upgrade job that validates no in-flight requests are pinned to interfaces scheduled for teardown.

Security & networking posture: new security group rules are additive and default-deny for external egress. We included a hardened subnet tagging convention (redwood:controlplane=true, redwood:ephemeral=true) to separate ephemeral vs. persistent allocations for auditors. The Terraform changes include controlled IAM permissions for the installer role to call DescribeNetworkInterfaces, CreateNetworkInterface, and DeleteNetworkInterface; least-privilege policies are documented in docs/privatelink/iam-boundaries.md.

Rollout plan and backwards compatibility: feature is guarded by a new installer flag --enable-ephemeral-enis (false by default). When disabled, behavior is unchanged. The migration job supports three modes: dry-run (log-only), migrate (move sticky endpoints to new AZ-bound allocation without switching traffic), and cutover (switch DNS/route associations). Rollback is supported by re-running the post-upgrade rollback job which re-applies previous PrivateLink allocation state from a snapshot stored in S3 or local state for air-gapped installs.

Testing and validation: unit tests for the allocator logic (Go) and Terraform unit checks added. Integration terratest suite exercises multi-AZ create/migrate/delete flows on a small sandbox account and asserts no traffic loss across a control-plane canary. Helm lint and kubeconform checks added. A new lightweight chaos test simulates ENI deletion to ensure automatic recovery.

Docs and ops: added detailed operational runbook docs/privatelink/op-runbook.md outlining how to force-evict an ENI, how to increase ephemeral_lifetime, and how to inspect DynamoDB allocation state. Also included upgrade notes and a sample Linear ticket template for field engineers.

Checklist: - [x] Terraform module + examples - [x] Helm chart changes + pre/post hooks - [x] Installer operator sidecar for allocation + GC loop - [x] Unit/integration tests - [x] Docs and runbook - [x] IAM least privilege guidance

Risks and mitigation: garbage collection is TTL-based and conservative by default (72h). During the first rollout, we recommend enabling the feature in a single region and monitoring metrics redwood.installer.enis.gc.count and redwood.installer.privatelink.failover.rate.

Related: builds on top of prior subnet and peering work; ties into ENG-1873 (controlled egress routing) and INF-302 (privatelink observability).
Diego Ramirez: Can we avoid DynamoDB for fully on-prem installs? Suggest a file-backed state option and a clear warning in docs.
Elena Park (reply): Added a file-backed adapter that uses local statefile under installer/state/ for air-gapped installs and documented differences. See installer/internal/allocator/file_store.go.
Maya Chen: Security review: IAM changes look OK but need an explicit audit log for ENI create/delete ops so SOC can correlate. Also request SSM parameter to toggle GC TTL in prod.
Elena Park (reply): Added cloudtrail event filter example and SSM param integration for dynamic TTL in installer config. Updated docs accordingly.
Liam O'Connor: Tests are comprehensive; minor nit - prefer 48h default TTL over 72h to reduce orphaned resources during long-lived test clusters.
Elena Park (reply): Agreed; lowered default in module variables to 48h and added a migration note in upgrade docs.
Add support for ephemeral control-plane ENIs and zone-bound PrivateLink allocations. Installer flag --enable-ephemeral-enis controls behavior. Includes Terraform module, Helm changes, installer allocator, and docs. Backward compatible when flag is disabled.
