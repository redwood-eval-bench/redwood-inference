# Add staged rollout variables by region (default: us-east + A100 only)

Internal: Terraform modules now support region/GPU-type rollout gating with safe defaults (only us-east + A100 enabled) for the provision-to-ready staged rollout.
Title: Add staged rollout variables by region (default: us-east + A100 only)

Context
As part of the provision-to-ready automation staged rollout (ENG-4829), we need Terraform defaults that prevent accidental enablement outside the initial canary scope. The control-plane/orchestrator feature flag can block later in the flow, but we want the IaC layer to enforce guardrails earlier by constraining which regions and GPU SKUs can be provisioned with the new pipeline hooks.

What this PR does
- Introduces module-level rollout gating variables keyed by region and GPU type.
- Defaults the rollout to us-east + A100 only (safe-by-default for week 4 canary).
- Adds consistent outputs that the provisioning orchestrator uses to determine whether a pool is eligible for staged provisioning.
- Updates validation to fail fast on invalid combinations (e.g., unknown region keys, empty allowlists when rollout is enabled).

Changes (high level)
- new variables:
  - staged_rollout_enabled (bool, default: true)
  - staged_rollout_allowed_regions (set(string), default: ["us-east-1"])
  - staged_rollout_allowed_gpu_types (set(string), default: ["a100"])
  - staged_rollout_mode (string, default: "allowlist")
- new locals:
  - effective_rollout_enabled = staged_rollout_enabled && contains(allowed_regions, var.region) && contains(allowed_gpu_types, lower(var.gpu_type))
- new outputs:
  - rollout_eligible (bool)
  - rollout_reason (string; empty when eligible)
- module wiring:
  - dedicated-pool module now passes rollout vars to submodules that emit tags/outputs consumed by the provisioning automation.

Files / modules touched
- modules/dedicated_pool/variables.tf
- modules/dedicated_pool/outputs.tf
- modules/dedicated_pool/main.tf
- modules/fleet_tags/main.tf
- envs/prod/us-east/dedicated/*.tfvars (example)
- docs/dedicated/rollouts.md

Plan output examples
Example A: us-east-1 + A100 (expected eligible)
- Input:
  - region = "us-east-1"
  - gpu_type = "a100"
- Output:
  - rollout_eligible = true
  - rollout_reason = ""
- Tags:
  - redwood:provisioning_rollout = "enabled"

Example B: us-west-2 + A100 (expected blocked by default)
- Input:
  - region = "us-west-2"
  - gpu_type = "a100"
- Output:
  - rollout_eligible = false
  - rollout_reason = "region_not_allowed(us-west-2)"
- Tags:
  - redwood:provisioning_rollout = "disabled"

Example C: us-east-1 + H100 (expected blocked by default)
- Input:
  - region = "us-east-1"
  - gpu_type = "h100"
- Output:
  - rollout_eligible = false
  - rollout_reason = "gpu_type_not_allowed(h100)"

Notes
- The allowlists are intentionally conservative and can be expanded by explicit env tfvars as we move to the next region/SKU.
- This is meant to complement (not replace) the control-plane feature flag / kill switch.

Testing
- terraform fmt / validate
- terraform plan (sample envs)
- added minimal unit-style checks via variable validation blocks (ensures non-empty allowlists in allowlist mode)

Rollout / Ops
- For the first staged rollout, no env overrides are required because defaults already scope to us-east-1 + A100.
- To expand rollout, update env tfvars:
  - staged_rollout_allowed_regions = ["us-east-1", "eu-west-1"]
  - staged_rollout_allowed_gpu_types = ["a100", "h100"]

Checklist
- [x] Safe defaults (constrained rollout)
- [x] Outputs documented for orchestrator consumers
- [x] Plan examples included
- [x] Reviewed with Fleet/Release for rollout sequencing

Review conversation (highlights)
Vanessa Ortiz: Can we ensure the default behavior is restrictive even if someone forgets to pass gpu_type in a module call?
Keiko Yamamoto (author): Yes — gpu_type is already required in dedicated_pool, and I added validation to error if it’s empty/unknown. Also lower() normalization ensures "A100" vs "a100" doesn’t bypass allowlist checks.

Connor O'Brien: Please add a short docs section that explains how this interacts with the control-plane feature flag so release comms don’t overpromise.
Keiko Yamamoto: Added docs/dedicated/rollouts.md with a note: Terraform gating prevents accidental provisioning; control-plane kill switch remains authoritative for runtime behavior.

Nina Petrova: Suggest making rollout_reason stable strings (no freeform) so tooling can key off it.
Keiko Yamamoto: Updated rollout_reason to use stable enums: region_not_allowed(<region>) and gpu_type_not_allowed(<type>) only.

Harish Reddy: Any risk this blocks emergency capacity adds during an incident?
Keiko Yamamoto: Defaults are restrictive, but capacity ops can override allowlists in tfvars with explicit change review. Also, setting staged_rollout_enabled=false preserves previous behavior for emergency cases.

CI
- terraform-lint: pass
- terraform-validate: pass
- docs-check: pass

Merge
Squash merged after approvals; no follow-up changes required for ENG-4829 beyond env-specific overrides as we expand rollout scope.
