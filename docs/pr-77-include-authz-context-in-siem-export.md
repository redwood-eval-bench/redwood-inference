# Include authz decision context in SIEM export (ECS/CIM mapping)

Summary
Adds support for exporting RBAC v2 authorization decision context fields (subject/action/resource/scope/policy_version/decision/denial reasons) through the audit-log-exporter and maps them to common SIEM schema conventions.

This PR is intended to unblock enterprise security questionnaire asks and pilot customer requirements where SIEM rules need to distinguish auth failures (deny) from application errors, and correlate access decisions with specific permission strings.

Context
- Redwood platform services are adding authz decision fields to audit log events (see redwood monorepo PR: redwood#28292).
- RBAC v2 requires evidence-friendly audit trails (SOC2 + enterprise questionnaires).
- Customers commonly ingest our audit logs into Splunk (CIM) and/or Elastic (ECS). They want stable field names, consistent typing, and predictable nullability.

What changed
1) New exported fields (non-breaking)
The exporter now forwards and normalizes these authz decision fields when present in the source event:
- authz.subject.id
- authz.subject.type (user | service_account | api_key | unknown)
- authz.subject.org_id
- authz.action (permission string, e.g., "org.api_keys.rotate")
- authz.resource.type (org | project | environment | deployment | cluster | api_key | audit_log_export | unknown)
- authz.resource.id (when available)
- authz.scope (canonical string representation: org:<id>[/project:<id>[/env:<id>]]; cluster scoped adds "/cluster:<id>")
- authz.decision (allow | deny)
- authz.denial_reason_codes (list of stable machine codes)
- authz.denial_message (optional, admin-only text; exporter defaults to NOT exporting free-form denial messages unless explicitly enabled)
- authz.policy.version (string)
- authz.policy.evaluator ("embedded" | "central" | "unknown")

2) SIEM schema mapping
Adds two output mappers:
- Elastic ECS-ish mapping (default for JSON output):
  - event.action = authz.action (when present)
  - event.outcome = success|failure (mapped from authz.decision)
  - user.id / user.type populated from authz.subject when the event actor is a human user
  - service.name = "redwood-audit-log-exporter"
  - labels.redwood_authz_policy_version, labels.redwood_scope (string labels for searchability)
  - tags includes "authz" when authz context exists

- Splunk CIM-friendly mapping (opt-in, for HEC payloads):
  - action = authz.action
  - status = allowed|denied
  - user / src_user populated when available
  - vendor_product = "Redwood Inference"
  - additional CIM-like fields namespaced under redwood.* to avoid collisions

3) Redaction / safety controls
- Denial message exporting is disabled by default (to avoid leaking internal policy evaluation details into customer SIEMs).
- Denial reason codes are exported as a list of stable codes only.
- Adds config flag: EXPORT_AUTHZ_DENIAL_MESSAGE=true to include authz.denial_message (intended for internal dogfood or Private installs where customers explicitly request it).

4) Compatibility behavior
- If upstream events do not yet contain authz context, exporter behavior is unchanged.
- If upstream events contain partial authz context, exporter exports what is present and avoids inventing data.
- No schema version bump required; this is additive.

Implementation notes
- Adds a small normalization layer in the pipeline that:
  - validates decision enums (defaults to "unknown" behavior by omitting outcome mapping)
  - coerces denial_reason_codes to [] if null
  - canonicalizes scope formatting to a stable string
- Ensures stable field types across outputs:
  - denial_reason_codes is always an array in JSON output
  - policy.version is always a string when present

Testing
- Unit tests for:
  - JSON mapping outputs for allow and deny decisions
  - Splunk HEC payload formatting with and without authz fields
  - redaction behavior for denial_message
- Golden fixtures:
  - testdata/events/authz_allow.json
  - testdata/events/authz_deny.json
  - testdata/expected/ecs_allow.json
  - testdata/expected/ecs_deny.json
  - testdata/expected/splunk_hec_deny.json

Example output (JSON / ECS-ish)
Input event (abridged): actor=user:123 action=org.api_keys.rotate decision=deny reasons=["MISSING_PERMISSION","SCOPE_MISMATCH"] policy_version="rbac-bundle@2025-03-14.1"

Exported record (abridged):
- event.action: "org.api_keys.rotate"
- event.outcome: "failure"
- user.id: "123"
- labels.redwood_authz_policy_version: "rbac-bundle@2025-03-14.1"
- redwood.authz.decision: "deny"
- redwood.authz.denial_reason_codes: ["MISSING_PERMISSION","SCOPE_MISMATCH"]

Files / areas changed
- src/pipeline/normalize_authz.ts (new)
- src/mappers/ecs.ts
- src/mappers/splunk_cim.ts
- src/config.ts (new flags)
- src/types/audit_event.ts (authz field typing)
- test/mappers/ecs.test.ts
- test/mappers/splunk_cim.test.ts
- testdata/events/*
- testdata/expected/*
- README.md (configuration + field reference)

Commits (squash summary)
1. types: extend audit event schema with optional authz decision context
2. pipeline: normalize authz fields and enforce stable typing
3. mappers: add ECS-ish mapping for authz fields
4. mappers: add Splunk CIM mapping + HEC payload keys
5. config: add EXPORT_AUTHZ_DENIAL_MESSAGE flag (default false)
6. tests: fixtures + golden tests for allow/deny cases
7. docs: document new fields + schema mapping notes

Review notes / discussion highlights
- Marcus Lin: requested denial_message be off by default and that we prefer reason codes. Implemented.
- Allison Grant: asked to keep existing top-level fields stable and avoid renaming legacy actor fields. Confirmed additive only; legacy actor.* remains unchanged.
- Logan Wright: suggested always emitting arrays for list fields to avoid schema drift in SIEM; updated normalization to force [] instead of null.

CI
- build: pass
- lint: pass
- unit: pass
- integration (splunk hec smoke): pass

Rollout / coordination
- This PR is intended to be deployed alongside redwood#28292 (platform emitting authz decision fields).
- No customer-visible change until upstream events include authz context.
- Follow-up: once RBAC v2 is enabled for pilots, we should confirm Splunk saved searches against event.outcome + event.action behave as expected (owner: Kevin Osei).
Audit log SIEM export now includes RBAC authorization decision context (allow/deny, permission, scope, policy version) when present in upstream events; additive and backward compatible.
