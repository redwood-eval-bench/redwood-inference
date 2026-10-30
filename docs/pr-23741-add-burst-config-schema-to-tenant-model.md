# Add burst config schema to tenant model

## Summary
Adds tenant config schema fields required for Dedicated Burst Capacity (enablement, caps, and priority tier). This PR is intentionally scoped to schema + migrations + validation only; no admission decisions or routing changes are included.

This unblocks ENG-4816 (control plane API to set burst config) and later work in the admission controller/routing paths.

## Motivation / Context
Dedicated burst capacity needs a durable, validated tenant-level configuration that is:
- explicit (opt-in per tenant)
- safely defaulted off for existing tenants
- constrained by guardrails (prevent accidental “infinite burst”)
- representable in both internal config (DB) and public/admin APIs (protobuf/JSON)

## What changed
- Tenant model extended with `burst_config` (nullable/optional) block.
- Added fields for:
  - `enabled`: bool (default false)
  - `max_burst_ratio`: float (baseline multiplier; e.g. 1.0–2.0)
  - `max_burst_gpus`: int (absolute cap; optional, used as a hard stop)
  - `burst_priority_tier`: enum (default STANDARD; allowed: STANDARD, SILVER, GOLD)
  - `cooldown_seconds`: int (optional; defaults set in policy engine; schema enforces bounds)
- DB migration to persist the new burst config fields.
- Validation rules + error messages (shared by control plane API + internal tooling).
- Added redaction notes for logging (avoid dumping full tenant config in error logs).

## Validation / Guardrails
Validation rules added in tenant config validator:
- `max_burst_ratio` must be between 1.0 and 3.0 (inclusive).
- `max_burst_gpus` must be >= 0; if 0 treated as “no burst” when enabled is false.
- `cooldown_seconds` must be between 0 and 3600.
- Priority tier must be one of the known enum values.
- If `enabled=true`, at least one of `max_burst_ratio` or `max_burst_gpus` must be set (to avoid unbounded borrowing).

Errors are returned as typed validation failures so the API layer can map them to clear user-facing messages.

## Backward compatibility
- Existing tenants: `burst_config` absent -> behaves as disabled.
- Migration sets defaults such that no tenant is implicitly enabled.
- No runtime behavior changes are included in this PR.

## Not included / Follow-ups
- Control plane endpoints to edit these fields (ENG-4816 will wire these into admin/customer config flows).
- Admission controller enforcement semantics (ENG-4810 / ENG-4821).
- Audit log events for burst config changes (tracked in ENG-4870 / separate PR).
- Billing/metering interpretation of burst usage (ENG-4827).

## Implementation notes
- Schema is represented in both protobuf and the tenant JSON config model; conversion functions added with strict enum mapping.
- Validator lives in the shared config package to be reused by API + CLI.

## Testing
- Unit tests for validator bounds, enum mapping, and defaulting.
- Migration test covers roll-forward + rollback.

## Screenshots
N/A (schema-only).

## Checklist
- [x] Migration is backward compatible
- [x] Defaults keep burst disabled
- [x] Validation errors are user-actionable
- [x] Unit tests added

---
### Commits (squash summary)
1) Add `BurstConfig` to tenant model + proto/JSON conversion helpers
2) Add DB migration + rollback
3) Add validator + tests; tighten logging

### Changed areas (high level)
- `controlplane/tenant/model/*`
- `controlplane/tenant/validation/*`
- `db/migrations/*`
- `api/admin/*` (types only; no endpoints)

### Illustrative snippet (validator error messaging)
Example failure returned by validator when enabled without caps: "burst_config.enabled requires max_burst_ratio and/or max_burst_gpus to be set".

---
## Review conversation (highlights)
**Ethan Park:** Can we ensure the default path for existing tenants doesn’t create an empty burst_config object that later code interprets as enabled?
**Benji Okafor (author):** Yep. I kept it optional/nullable end-to-end; conversion code only materializes it when present. Added a test case for absent burst_config.

**Jordan Lee:** Please align ratio bounds with policy engine expectations. I think we said 1.0–3.0 for schema, but policy may clamp to 2.0 by default.
**Benji Okafor:** Updated schema validator to 1.0–3.0. Policy can clamp further via config rules; schema just prevents foot-guns.

**Cole Summers:** We’ll need stable field names for metering export. Can you confirm naming won’t change?
**Benji Okafor:** Confirmed. Added comments noting that billing/metering will key off these fields; any renames will require migration + comms.

**Naomi Feldman:** Please avoid logging full tenant configs on validation errors; we’ve had issues with config dumps in logs.
**Benji Okafor:** Good catch. Switched error logging to include tenant_id + field name only; added a regression test for redaction.

Approvals:
- Ethan Park: Approved
- Jordan Lee: Approved (after bounds update)
- Cole Summers: Approved
- Naomi Feldman: Approved

---
## CI
- build (bazel): pass
- unit tests: pass
- lint: pass
- migration tests: pass

## Merge
Merged via squash into `main`.
