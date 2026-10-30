# Integrate terraform state checkpoints and helm canary rollback hooks for safer private upgrades

Motivation: Customers running Redwood Private reported a painful manual recovery process when an in-place upgrade diverged state between Terraform-managed infra and Helm-managed application state. This PR combines lightweight checkpointing of Terraform state with Helm-level canary/rollback hooks and an end-to-end restore verification step. The goal is to make upgrades reversible and auditable without forcing full cluster snapshots for every minor release.

What this PR does (high level): 1) Adds a Terraform state checkpointer that emits time-stamped, KMS-wrapped state checkpoints to the configured object store before apply. Checkpoints are retained according to the existing retention policy. 2) Adds Helm chart lifecycle hooks for a canary phase: pre-upgrade validation job, post-upgrade verification job, and a rollback trigger that can be executed by the installer if verification fails. 3) Introduces a restore verifier command in the private installer CLI that performs a dry-run restore of the checkpointed state into a temporary namespace and validates core control-plane liveness and DB schema invariants. 4) Wires a coordinated upgrade workflow that will attempt a canary release, run verification, and only then promote the release cluster-wide; on failure it will auto-trigger the Terraform checkpoint restore and Helm rollback.

Details & design decisions: - Checkpoint format: state files are exported as compressed JSON and signed with a HMAC using a key derived from the customer's KMS config. We avoid full snapshots for speed; checkpoints focus on Terraform state + critical installer manifests (CRDs, release metadata). - Retention & GC: default retention 7 checkpoints; configurable via installer config; GC runs with locks to avoid races. - Helm hooks: canary label applied to 10% of control-plane pods (configurable). pre-upgrade job verifies storage ACLs and image pull secrets; post-upgrade job runs smoke tests and DB migration sanity checks. - Restore verifier: creates a namespaced clusterrolebinding-scoped temporary environment to validate that applying the checkpoint restores expected object counts and that DB migrations that were part of the release have idempotent down/up behavior. - Safety guard: an escape hatch parameter allow_manual_rollback=false by default for new installs; existing installs keep prior behavior.

Testing & rollout: Full e2e in k8s-kind (small) and in a replicated QA cluster that mirrors customer topology (AWS VPC with private subnets). Tested failure scenarios: failed image pulls, partial migration failure, object-store write failures. We validated that automatic rollback completes within configured SLA in 98% of trials and that checkpoints are restorable in all manual recovery runs.

Breaking changes & migration: No breaking APIs. Default behavior is opt-in for older clusters (installer detects pre-2024.10 installs and preserves prior flow).

Follow-ups: Add a UI toggle in the Console to visualize checkpoints and initiate manual restores (ENG-4152). Add an analytics event for checkpoint creation (ENG-4241).
Maya Thompson (2024-11-07): Can you clarify the retention GC locking to avoid races with concurrent manual GC? Also suggest a unit test for the HMAC signature code.
Sofia Patel (2024-11-07): Added lock file semantics and unit tests for HMAC; pushed an update that changes object prefixing to include cluster-id.
Ethan Garcia (2024-11-08): The canary percentage should be configurable at runtime; please expose an env var and document the default.
Sofia Patel (2024-11-08): Exposed CANARY_PERCENT and documented in values-schema.yaml. Ran the canary jobs locally and added e2e asserts.
Liam Chen (2024-11-11): Verified the e2e logs — rollback trigger fired correctly on DB migration failure. Approving pending a small doc tweak.
Maya Thompson (2024-11-12): Approve — nice job adding restoration verification; flagged retention default for product to adjust later.
feat(checkpoint): add terraform state checkpointer and uploader
feat(helm): add canary pre/post verification jobs and rollback hook
feat(cli): restore-verifier command and dry-run restore flow
chore(tests): add e2e scenarios for checkpoint restore, failure injection
docs: document new upgrade flow and config knobs
fix: tighten GC locking around checkpoint retention
Adds upgrade safety automation: terraform state checkpoints, Helm canary hooks, and a restore verifier CLI. Default retention 7 checkpoints; opt-in for pre-2024.10 clusters.
Unit tests for checkpoint signer/uploader and GC locks. e2e scenarios cover successful canary promote, canary-driven rollback, object-store failure, and manual restore. QA replicated cluster used to validate timing/SLA; observed median auto-rollback 2.8m, 95th 6.4m in injected failure tests.
1) Installer creates TF checkpoint and uploads to <bucket>/<cluster-id>/checkpoints/<ts>.tgz; 2) Helm does canary deploy to labeled subset; 3) Run canary pre/post jobs; 4) If verification passes, promote; else invoke 'installer restore --checkpoint <id>' and 'helm rollback <release>'
