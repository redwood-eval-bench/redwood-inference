# AZ-aware egress scheduler and endpoint-binding rollback workflow

Motivation: Private control planes deployed into customer VPCs were occasionally incurring cross-AZ egress and uneven endpoint affinity during scale events. This PR introduces an AZ-aware egress scheduler plus a safe endpoint-binding rollback workflow so that control plane endpoints and egress proxies are assigned deterministically to availability zones and can fail over cleanly when bindings fail.

Summary of changes: adds a lightweight Terraform scheduler module that computes AZ-local egress endpoint assignments at apply-time, updates the subnet-handoff module to include az_affinity tags, extends the Helm installer with values to opt into AZ-local egress, and provides an operational rollback script that unbinds endpoints and performs staged failover to a shared egress path. Also includes integration tests and documentation.

Why this approach: Deterministic assignment reduces cross-AZ data transfer and makes routing predictable for latency-sensitive control plane traffic. Doing the assignment at provisioning time keeps runtime complexity low and avoids coupling the control-plane agents to a central allocator. The rollback workflow is intentionally conservative: it unbinds problematic endpoints, verifies connectivity from a canary subnet, and then promotes a fallback egress.

Upgrade notes / migration: Existing installs are opt-in. To adopt, upgrade the installer Helm chart and re-run terraform apply with the new module enabled (see examples/private/az-egress-vars.tf). For production fleets, we recommend a staged rollout (enable on a single region, monitor latency/cost for 24h, then enable region-wide). The PR includes a runbook in docs/installer/az-egress.md with the rollback commands.

Testing: terraform fmt/plan checks, unit tests for the scheduler logic, full e2e in sandbox (simulated 3 AZs) validating affinity behavior and failover, and manual verification with live VPCs (redwood-sandbox).

Checklist: - [x] Terraform module + docs - [x] Helm values + chart bump - [x] Integration tests - [x] Rollback script and runbook - [x] CI green

Related: ENG-4521, SUP-8810. No API changes to public Redwood control plane; this is installer-only.
Diego: Please ensure the scheduler output is idempotent across consecutive applies — observed drift in sandbox. Suggested adding stable naming with subnet id + az suffix.
Aisha: Addressed — switched to deterministic name format using subnet_id + '-egress' and added extra acceptance test to catch drift.
Maya: The rollback script is helpful but can be dangerous if run accidentally. Can we add a dry-run mode and explicit confirmation?
Aisha: Added --dry-run and a confirmation prompt; updated runbook to call dry-run as first step.
Jonah: CI flake on e2e where the canary verification timed out at 60s. Recommend increasing check timeout and adding retry loop.
Aisha: Increased timeout to 180s and added exponential backoff for health checks. Re-ran CI and green.
Introduce AZ-aware egress scheduler and safe endpoint-binding rollback workflow. Opt-in installer changes allow deterministic AZ-local egress endpoints, reduced cross-AZ egress costs, and an operational rollback script. See docs/installer/az-egress.md for migration and runbook.
Run terraform apply with module enabled and monitor per-AZ egress bytes in console. If a binding fails, run installer/scripts/endpoint_rollback.sh --dry-run first, then without --dry-run to perform unbind and failover. Recommended staging: enable in one region for 24h before wider rollout.
