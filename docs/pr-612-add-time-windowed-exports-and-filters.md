# Add time-windowed exports + actor/event-type filters (report templates)

## Summary
This PR enhances `audit-log-exporter` to support compliance/evidence workflows by adding (1) explicit time-windowed exports and (2) optional filters for actor + event types. It also introduces a small report-template layer so the compliance reporting pack can request a known preset ("last_90_days_admin_activity", "rbac_changes", etc.) without re-encoding filter logic in multiple places.

Motivation: enterprise security reviews are consistently asking for "audit logs for a defined period" and "only admin/security-relevant events". We also had a support report (SUP-3917) where customers believed exports were missing event types; part of that was unclear defaults and lack of an explicit event-type filter list in the export request.

## API changes
### New request fields (non-breaking)
- `start_time` (RFC3339, inclusive)
- `end_time` (RFC3339, exclusive)
- `actor_ids` (optional array of strings)
- `actor_types` (optional array: `user|service_account|api_key`)
- `event_types` (optional array of strings; validated against known types)
- `template_id` (optional string; mutually exclusive with providing filters directly unless `allow_override=true`)

Behavior:
- If `start_time`/`end_time` are omitted, behavior remains "export all available within retention" (existing behavior).
- If provided, exporter enforces the window at query time and re-checks in stream/iteration to avoid boundary drift.
- `event_types` is an allowlist filter; unknown types return 400 with a list of valid types.
- `actor_ids` and `actor_types` are ANDed with the time-window and event filter.

### Report templates
Adds `templates/*.yaml` defining a stable set of presets that map to filters + default time windows. Example preset:
- `rbac_changes_last_180d` -> `event_types=[rbac.role.created, rbac.role.updated, rbac.binding.created, rbac.binding.deleted]`, window=180d

Template contract:
- Templates are versioned by exporter release; template IDs are stable.
- Export response includes `applied_filters` so customers/auditors can see exactly what was used.

## Implementation notes
- Query layer updated to accept a `FilterSpec` that compiles to SQL WHERE clauses + parameter binding.
- Streaming export code now emits a header block with: `export_id`, `generated_at`, `window`, `filters`, `template_id` (if any).
- Added a canonical "known event types" registry and tests ensuring registry coverage matches the producer schema we expect.
- Pagination: time-windowed exports use `(timestamp, id)` cursor to avoid duplicates when multiple events share the same timestamp.

## Security/Compliance considerations
- Filters are validated server-side (no raw SQL fragments; strict param binding).
- `actor_ids` is limited to 200 entries to cap query size.
- Maximum export window defaults to 365 days unless `--allow-large-windows` is enabled (intended for internal GRC workflows only).
- Added explicit audit event for `audit_log.export.generated` that records: caller identity, window, filters, and bytes written.

## Testing
- Unit tests: filter compilation, RFC3339 parsing, template resolution, unknown event-type validation.
- Integration tests: seeded audit events across boundary timestamps; verifies inclusive/exclusive semantics.
- Load sanity: ran export for 90d in staging with 1.2M rows; query time stable and memory constant in streaming mode.

## Rollout
- Default behavior unchanged. Consumers can adopt templates incrementally.
- Follow-up: wire `template_id` usage into compliance report pack generator (ENG-18441) once this merges.

## Example request
`POST /v1/exports`
{
  "template_id": "rbac_changes_last_180d",
  "start_time": "2024-08-01T00:00:00Z",
  "end_time": "2025-02-01T00:00:00Z"
}

## Checklist
- [x] Backwards compatible
- [x] Tests added
- [x] Docs updated (README + examples)
- [x] Security review notes captured (filters + limits)
- [x] Added exporter-side audit event for export generation
Marcus Lin: Please confirm window semantics (inclusive/exclusive) are documented and tested, and ensure we’re not vulnerable to massive `actor_ids` payloads. Also: do we need an exporter audit event for generating an export?
Kevin Osei (author): Documented in README + added integration tests for inclusive start/exclusive end. Added max 200 actor IDs and a request size cap. Implemented `audit_log.export.generated` event emitted on successful export start (includes window/filters/template).
Olga Petrov: Template IDs need to be stable and customer-safe. Can we include the applied filters in the export output so auditors don’t have to trust our template mapping?
Kevin Osei (author): Added `applied_filters` header block (JSON) at the start of the exported artifact and also in the API response metadata. Template IDs are now validated against a registry and treated as public identifiers.
Sean Gallagher: Cursor should be deterministic. Using timestamp-only can duplicate/skip on high volume; recommend (timestamp,id) ordering and ensure we have an index. Also add guardrails for >365d windows.
Kevin Osei (author): Updated pagination to order by `(timestamp, id)` and added/verified index in migrations. Added 365d default max window with an explicit server flag for internal workflows.
