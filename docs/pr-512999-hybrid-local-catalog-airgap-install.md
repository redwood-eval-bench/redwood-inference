# hybrid local catalog and registry-fallback for air-gapped installer

Adds a hybrid local catalog mode for air-gapped installs that uses a lightweight index plus preseeded artifact bundles; improves installer reliability when customer registries are intermittently unreachable.
Motivation:
Many on-prem customers run registries behind strict network controls. The current airgap installer requires a full OCI registry seed or a fully preseeded tarball. Both have tradeoffs: full seeding is heavy for frequent updates; tarballs lack fine-grained integrity checks and incremental upgrade support.

This PR introduces a hybrid local catalog mode: at install time the customer can choose to seed a minimal index (catalog JSON) and a small set of verified OCI blobs. The installer can then lazily fetch additional blobs from a sealed bundle via a pull-through staging registry or fall back to a bundled tarball when the registry is unreachable. Integrity is enforced with a signed catalog and SHA256 checksums. The mode reduces initial bandwidth and makes upgrades incremental while preserving air-gap safety guarantees.

What changed (high level):
- installer/pkg/catalog: new catalog index format, signing/verify helpers, and catalog->bundle reconciliation logic.
- charts/airgap-preload: Helm pre-install hook that validates catalog signature and generates prefetch plan.
- terraform/modules/registry_sync: new optional resource to deploy a lightweight staging registry (readonly) and associated ACLs for customers who want a local mirror without full seeding.
- tools/airgap/pack: new packer command `hybrid-pack` that produces a small index JSON + minimal blobs tar that the installer understands.
- docs/airgap/README.md: updated with operator workflow, troubleshooting, and upgrade/rollback notes.

Checklist:
- [x] catalog format defined and documented
- [x] signer + verifier implemented (KMS/HSM compatible interface)
- [x] Helm hook validates and writes prefetch job to node-local cache
- [x] Terraform module optional, doesn't force usage for existing customers
- [x] Integration test that simulates registry unreachable and verifies fallback to tarball

Testing and validation:
- Local integration test (ci/airgap/hybrid) simulates partial seed, then disables registry and runs installer; verified node prefetch and fallback path.
- E2E in the lab: deployed to k3s with limited disk/IO to exercise incremental pulls and verify KV cache behavior under constrained conditions.
- Security review: signer/verifier audited for predictable canonical JSON and replay protections.

Rollback and upgrade notes:
- Hybrid mode is opt-in; existing full-seed and tarball workflows unchanged.
- Upgrades: installer reconciles catalog and will prefer staged registry blobs if checksums match. If a new catalog cannot be validated, the installer refuses the incremental upgrade and suggests a full tarball apply.

Migration path for customers:
- Provide `hybrid-pack` to generate initial catalog + minimal blobs. Customers can later seed a staging registry using terraform/modules/registry_sync. The installer includes a one-time registry-attach step.

Related work:
- ENG-2345: incremental vendor bundles (design review)
- ENG-5901: catalog signing and verifier interface
- SUP-7782: customer ticket requesting lower initial seed size for regulated environment

Notes for reviewers:
- The catalog index format includes a version field; we expect future expansion for per-blob storage hints.
- Performance: catalog verification is cheap (streaming SHA256 over manifest), prefetch plan generation is O(N) where N is minimal blobs (usually < 200).

Files of interest and usage examples are listed below in files_changed.

Graham Li: Left a couple of suggestions on catalog canonicalization to avoid signature breakage across platforms. Also asked for a test that simulates a tampered catalog.
Asha Menon: Responded explaining canonical JSON library choice and added a tamper-detection unit test in e9f0a11.
Marisol Ortega: Requested clarity around RBAC for the staging registry; suggested an optional read-only service account and documented the ACLs in terraform module.
Evan Stumpf: Noted a potential race in the Helm hook when prefetch job starts before node-local cache is writable; requested an initContainer to ensure directory permissions.
Asha Menon: Added an initContainer and updated job template (ff12ee3). Verified in lab that race is resolved.
