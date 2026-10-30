# Introduce KMS Introspector, Residency Soft-Guard, and Audit Obfuscator

Motivation: We saw two incidents where KMS selection mismatches caused requests to be routed to non-preferred regions, and audit logs exposed full service-principal identifiers in downstream systems. This PR adds three coordinated changes: 1) a lightweight KMS Introspector to fetch and cache key metadata (region, provider, tags) for decisioning; 2) Residency Soft-Guard, a policy shim that enforces residency decisions but allows configured soft fallbacks with enriched audit annotations; 3) Audit Obfuscator, an opt-in audit layer that pseudonymizes operator/service principal IDs using a per-tenant salt stored in the secrets store.

Summary of changes (single continuous narrative): The Introspector exposes an internal read-only API (kms_introspector.v1) used by the runtime routing layer and by the policy-enforcer to determine key residency claims without performing decrypt/encrypt operations. Residency Soft-Guard plugs into the RBAC v2 policy evaluation pipeline: when a request would violate a strict residency rule, Soft-Guard will consult fallback profiles (configured per-tenant) and, if a compatible fallback exists, allow the request with an audit annotation explaining the substitution. If no fallback exists and the tenant is in strict mode, the request is rejected. Audit Obfuscator runs as a final step before log export: it replaces full subject identifiers with salted pseudonyms and emits a keyed mapping to the internal audit vault accessible only to security ops. The mapping is HMAC-SHA256(salt, subject) with tenant salt rotation support.

Checklist:
- [x] KMS Introspector service and cache
- [x] Residency Soft-Guard policy shim and config schema
- [x] Audit Obfuscator with per-tenant salt rotation
- [x] Integration tests for policy decision paths
- [x] End-to-end canary in staging with simulated cross-region keys
- [x] Documentation and config examples for operators

Commits (high-level):
- "add kms introspector service and caching layer"
- "introduce residency soft-guard shim into rbac v2 pipeline"
- "implement audit obfuscator and mapping vault client"
- "add feature flags and operator config for fallback profiles"
- "integration tests and docs"

Changed areas (high level):
- services/kms_introspector/* — new read-only service plus cache adapter
- runtime/policy/enforcer/softguard.go — policy shim logic and config parsing
- runtime/audit/obfuscator.go — obfuscation pipeline and vault client
- api/internal/registry.proto — added kms_introspector.v1 RPC
- docs/compliance/residency_softguard.md — operator guide
- helm/values.yaml — flags to enable/disable features per-environment
- tests/integration/residency_softguard_test.go — simulated scenarios

Small illustrative snippet (policy decision annotation):
- "audit_annotation: 'residency_fallback=eu-west-1 -> eu-central-1;fallback_profile=geo-compatible;reason=operator-override-allowed'"

Review conversation (condensed as single stream):
- Marco Li (reviewer): "Nice separation of concerns. Two asks: (1) can we make cache TTL tenant-configurable? (2) please add a stronger guard when the mapping vault fails — we must avoid leaking unmapped IDs into exported logs."
- Aisha Patel (author): "Updated cache to be tenant-configurable and plumbed a fail-closed behavior for the mapping vault. Implementation note: when vault is unavailable and tenant is strict, we reject export; when soft mode, we emit 'subject=unmapped-placeholder' plus alert to ops."
- Grace Nguyen: "Integration tests look good. One nit: add a small comment documenting the HMAC key rotation procedure." 
- Daniel Kim: "Approved after minor nits resolved. Verified CI rerun passes."
Adds a KMS metadata introspector service, a residency soft-guard policy shim that can fallback to compatible key pools when strict residency enforcement is not configured, and an audit obfuscator that pseudonymizes operator/service principal identities while preserving traceability for security analysts.
Feature flags: helm.values.features.kmsIntrospector (bool), features.residencySoftGuard.mode (strict|soft). Default: disabled in stable; enabled in canary. Operators must provision per-tenant salt in secrets/vault/tenant-salts/<tenant-id>. Rolling rotation requires replay window setting to avoid mapping mismatches.
Stage: deploy with helm values enabling kmsIntrospector and residencySoftGuard.mode=soft. Run tests/integration/residency_softguard_test.go against mocked KMS endpoints. Validate audit exporter emits pseudonyms and that mapping vault entries are created. For strict tenants, assert requests are rejected when no compatible key found.
Introspector relies on KMS metadata tags; legacy keys without region tags will be treated as 'unknown' and require operator remediation. Audit mapping vault is an internal scoped secret — if that service is unavailable, exports behave as configured per-tenant (fail-closed for strict, placeholder for soft).
Planned follow-ups: (1) UI in Console to visualize fallback decisions and mapping lookup traces (ENG-2380); (2) add automated migration tool to backfill missing key-region tags (SEC-130); (3) evaluate adding cryptographic blinding to pseudonyms to support cross-tenant correlation under consented forensics flow.
Marco Li: request tenant-configurable cache TTL and fail-closed mapping vault
Aisha Patel: implemented changes and added tests
Grace Nguyen: small comment re: HMAC rotation added
Daniel Kim: approved
