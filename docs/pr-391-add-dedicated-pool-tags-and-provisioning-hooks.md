# Add dedicated pool standard tags + outputs for provisioning automation (and required var guardrails)

Context
Provision-to-ready automation needs a stable contract from IaC so the orchestrator can reliably discover: (a) which nodes belong to which dedicated pool, (b) what image channel/version the pool is pinned to, and (c) which regions/pool types have provisioning automation enabled. Today we have a mix of ad-hoc tags and inconsistent outputs across modules, which has made early rollout debugging slower than it needs to be (see SUP-28411 for the most recent "where did these nodes come from" escalation).

What this PR changes
1) Standardizes tags on dedicated GPU pools
- Adds a required baseline set of tags to the dedicated pool and node group resources:
  - redwood:fleet_pool_id (string)
  - redwood:fleet_pool_type (= "dedicated")
  - redwood:region (e.g. "us-east-1")
  - redwood:image_channel ("candidate" | "stable")
  - redwood:image_version (optional, when pinned)
  - redwood:provisioning_hooks_enabled ("true" | "false")
- Tags are applied to the backing ASG/instance template resources where supported so downstream discovery (agent + control plane) can use a single lookup path.

2) Adds outputs consumed by the provisioning orchestrator
- Exposes consistent module outputs:
  - pool_id
  - pool_region
  - image_channel
  - image_version
  - provisioning_hooks_enabled
- These outputs are intended to be wired into the fleet provisioning orchestrator config and the readiness service registration step.

3) Guardrails for required variables (fail fast)
- Adds validation/preconditions so plans fail early if:
  - pool_id is empty
  - region is unset/invalid
  - image_channel is not one of {candidate, stable}
  - provisioning hooks are enabled but image_channel is unset
- Keeps defaults non-disruptive: provisioning hooks remain opt-in via enable_provisioning_hooks (default false). Existing dedicated pools should see tag-only changes unless explicitly opting into hooks.

Notes / design choices
- Tag names intentionally do not include per-node identifiers (avoids metric/tag cardinality explosions downstream).
- We keep the tag namespace as redwood:* to match internal conventions and allow filtering/permissions rules in cloud consoles.
- image_version is optional to support channel-following pools (candidate/stable) without forcing a pin; orchestrator can treat (channel, version?) as the image selector.

Testing
- terraform fmt
- terraform validate
- tflint
- terratest: modules/dedicated_pool (basic plan/apply in ephemeral env)

Rollout
- Safe to merge: tags are additive. Outputs are additive.
- Teams enabling hooks should do so behind the existing region/pool feature flags in the provisioning automation rollout plan. Follow-up wiring will happen in control-plane/orchestrator PRs.

Checklist
- [x] Tags applied at the correct resource level (ASG/launch template where possible)
- [x] Outputs documented in modules/README
- [x] Validations don’t break existing callers (defaults preserved)
- [x] Reviewed with Fleet + Release Eng

Follow-ups
- Wire module outputs into orchestrator config generation (ENG-4822/ENG-4829)
- Update dashboards to rely on standardized tags for pool grouping once rollout expands beyond us-east A100
- Dedicated pool modules now emit standardized tags and outputs used by provision-to-ready automation (pool id, region, image channel/version). - Guardrails added to fail fast when required variables are unset or invalid (prevents silent drift and post-provision surprises).
Vanessa Ortiz: Can we make sure tags land on the instance template/ASG, not just the EKS nodegroup wrapper? The validation agent does cloud-side discovery early in bring-up.
Keiko Yamamoto: Good call. Updated gpu_node_group module to propagate tags to launch template and ASG tags (where supported). Added terratest assertions for both paths.
Jamal Pierce: pool_id needs to be stable and match what capacity planning uses. Can we keep the tag key name aligned with the readiness service fields? (fleet_pool_id vs pool_id)
Keiko Yamamoto: Renamed to redwood:fleet_pool_id for the tag and kept output as pool_id. Added a short note in README mapping tag->output->readiness field.
Sahana Iyer: Please keep enable_provisioning_hooks default false; we can’t risk enabling hooks broadly before the image pipeline is fully staged.
Keiko Yamamoto: Confirmed. Default remains false and the validation only triggers when explicitly enabled.
Fiona McCarthy: Any tags exposing customer identifiers? Also, ensure we’re not outputting anything that should be treated as secret.
Keiko Yamamoto: No customer identifiers added. pool_id is an internal UUID-like identifier; no secrets in outputs. Added a comment to avoid embedding customer names in pool_id values.
Harish Reddy: Minor nit: image_channel validation should error clearly (we see lots of "invalid value" with no hint). Can we improve the message?
Keiko Yamamoto: Updated variable validation error_message to include allowed values and an example.
