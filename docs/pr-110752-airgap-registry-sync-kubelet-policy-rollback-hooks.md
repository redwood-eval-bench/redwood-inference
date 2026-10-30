# add-airgap-registry-sync-and-kubelet-selinux-policy-with-atomic-rollback-hooks

Motivation: customers running on-prem/private installs have reported two recurring problems in highly-restricted environments: (1) inconsistent image availability when a local registry mirror drifts from the vendor lockfile, and (2) kubelet SELinux denials after runtime upgrades that break node bootstrap and KV cache preloads. This PR introduces a small, opinionated automation bundle to make offline registry mirrors deterministic and to ship a minimal SELinux policy + kubelet init hook that prevents common denials.

What this change does (single-string summary): adds a registry-sync Terraform module and controller job to generate deterministic OCI index snapshots, a signed vendor-lockfile manifest, a kubelet SELinux policy module with an installation hook, and atomic upgrade/rollback hooks (preload + prune + rollback) to keep nodes in a consistent state in air-gapped upgrades. Also adds Helm chart values and an optional sidecar that runs a pull-through cache reconciler.

Checklist: 
- [x] terraform module: registry-sync with OCI index export/import support and checksum verification
- [x] vendor-lockfile: deterministic manifest generator (sha256 list + image refs + arch variants)
- [x] kubelet policy: minimal SELinux module and systemd hook to load during bootstrap
- [x] upgrade hooks: preload-images, snapshot-kv-cache, and atomic rollback script for on-failure path
- [x] Helm: values.airgap.yaml with sidecar toggles and pull-through cache config
- [x] e2e/airgap tests: local k3s + mocked registry to validate end-to-end image sync and node boot path

Testing notes: ran CI matrix locally on a GHA runner configured to emulate an air-gapped environment. e2e used a small VM fleet (3 nodes) with containerd and SELinux enabled. Validated: node boots with preloaded images, kubelet SELinux AVCs are absent for the reported codepaths, registry-sync applies identical OCI manifests across mirrors, and rollback script restores previous image set and restarts kubelet in <45s for these test nodes.

Compatibility and rollout: this is additive and guarded by Helm values and Terraform opts; default is disabled for hosted installs. For customers using Redwood Private, ops can enable at time of install/upgrade by toggling values.airgap.enable_registry_sync and values.airgap.selinux_hook. Documented upgrade steps included in release notes.

Related work items: Linear ENG-34, ENG-182, PM-97.

Operational notes: registry-sync produces a signed manifest blob (vendor-lockfile.json.sig) that the installer verifies before applying; signatures are created using the customer's GPG key (optional) or a release key. The atomic rollback hook records the prior vendor manifest + kubelet preload snapshot under /var/lib/redwood/airgap/snapshots and will revert containerd overlays and restart services if post-upgrade health checks fail.


Ravi Patel: Left a few nits in the kubelet policy - prefer narrower allow rules and add rationale comments.
Maya Chen: Updated policy with narrower rules and inline justification in the .te file.
Aisha Gomez: Can we make the signed manifest verification optional? Some customers cannot store GPG keys in their vaults.
Maya Chen: Added flag values.airgap.verify_vendor_manifest; default true for enterprise builds, can be overridden in values.airgap.
Tom Rieder: CI e2e failed on first run due to race in preload script. Suggest adding a readiness probe for containerd before image preload.
Maya Chen: Added wait-for-containerd utility and retried preload with exponential backoff. Reran e2e; passed.
