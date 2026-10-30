# Introduce RBAC policy-presets, KMS provider plugin interface, and adaptive audit-sampling

Motivation: Customers with strict regulatory boundaries repeatedly request a) curated RBACv2 policy presets for common compliance postures (SOC2, GDPR data-local operators, restricted-operator), b) a stable plugin surface for KMS provider integrations (so private deployments can add internal HSM/KMS implementations without forking core), and c) better audit telemetry that reduces PII exposure while keeping forensic fidelity. This PR implements three coordinated changes: 1) policy-presets: a set of validated, parameterizable RBACv2 policy templates and a small CLI to materialize them for tenant bootstrapping; 2) kms-plugin: a lightweight provider interface and shim layer with two example adapters (aws-kms-compatible shim for Hosted tests, and a local hsm-mock used by our private-installer tests); 3) audit-sampling: adaptive audit sampling and content redaction rules driven by policy labels and risk tiers so we can sample full payloads only for sensitive operator actions and otherwise store compact event signatures.

Scope and user-visible behavior: The API, runtime, and console behavior is unchanged by default. Admins can opt into the presets via the new admin CLI (bin/rbac-presets apply --preset=restricted-operator --tenant=acme). The KMS plugin is an opt-in dependency for Private install and Dedicated operator-hosted deployments: a new extension point (plugins/kms/v1) accepts a go plugin or a containerized adapter (see docs/integrations/kms/README.md). Audit sampling is controlled by the new policy annotation audit.sampling: {mode: adaptive|always|none, sample_rate: 0.01}.

Design highlights: - Policy-presets are authored as parameterized YAML with constraints validated against the policy-compiler; templates live in policy/presets/*.yaml and include tests under policy/presets/testdata. - KMS plugin surface defines three operations (Wrap, Unwrap, Sign) and a minimal config schema; the runtime loads adapters at startup when configured and falls back to default internal KMS for hosted. - Adaptive audit-sampling computes event risk score using operator role, resource sensitivity label, and KMS provider confidence; high-risk events are stored with full payloads, low-risk events use deterministic hashes and compact metadata.

Backward compatibility and migration: Existing tenants see no change unless admins apply presets or configure a plugin. The policy-compiler validates imported presets and will reject incompatible templates; a migration helper (bin/rbac-presets lint --in=old-policy.yaml) is provided. The KMS plugin uses a strict versioned interface; older adapters must implement a compatibility shim.

Testing and rollout: Unit tests added for policy compilation, plugin loader, and audit sampler. End-to-end smoke tests added to redwood-private-installer CI (integration/kms/e2e). We ran a canary rollout to Dedicated where audit sampling was enabled for 10% of operator actions for two weeks; no regressions observed.

Checklist: - [x] Policy templates with tests - [x] CLI helper and linting - [x] KMS plugin interface + two adapters (aws-test shim, hsm-mock) - [x] Audit sampler + redaction rules - [x] Documentation updates (docs/integrations/kms/ and docs/operator/rbac-presets.md) - [x] E2E integration tests for private-installer

Related: ENG-2319 implements the admin UX follow-up; SUP-4421 captured a customer incident that motivated adaptive sampling. For discussion and design rationale see internal doc: https://linear.app/redwood/issue/ENG-2319/rbac-presets-and-kms-plugin (internal).
Diego Torres (reviewer): Please add a migration note for tenants with custom role templates; their role IDs may collide with preset names.
Author (Aisha Malik): Added lint step and explicit name-spacing to templates to avoid collisions; updated description.
Maya Chen (reviewer): The plugin loader looks fine, but we should fail fast on version mismatch rather than falling back silently.
Author: Loader now validates semantic version and returns a startup error if configured plugin is incompatible. Added unit test.
Omar Rahman (security reviewer): Audit redaction looks good but add a short-term override to force full capture for incident investigations.
Author: Added admin API endpoint: POST /admin/audit/override with TTL; documented in docs/operator/rbac-presets.md.
Maya Chen: Approved overall after requested changes.
Diego Torres: Approved.
Omar Rahman: Approved with comment on monitoring (see release notes).
Add policy-presets for quick tenant bootstrapping, a pluggable KMS provider interface with example adapters, and adaptive audit-sampling with configurable redaction policies. Opt-in features for Private/Dedicated. Admin CLI and migration helpers included.
