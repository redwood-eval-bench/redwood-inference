# router: add FPE decision tracing + enforce pinned-build fallback guardrails

## Context
During the 2025-02-02 us-east capacity event (SUP-4821) we had to manually answer whether pinned customers can be silently moved to a different model variant. We also lacked consistent reason codes for fallback decisions.

This PR adds FPE decision tracing + hard guardrails for pinned builds.

## What changed
- Adds `fallback_policy_engine` reason codes:
  - `primary_5xx`
  - `primary_timeout`
  - `p95_latency_with_high_util`
  - `manual_degrade`
- Adds a trace span `router.fpe` with attributes:
  - `fpe.enabled`
  - `fpe.reason`
  - `fpe.primary_route`
  - `fpe.selected_route`
- Enforces pinned build guardrail:
  - If `account.model_version_pinned == true`, disallow model-variant fallback. Region fallback remains allowed if within allowed region set.
- Adds a log line (INFO) whenever a fallback selection is applied (one per request, sampled).

## Files / areas
- `router/src/fpe/engine.rs`
- `router/src/fpe/compat.rs`
- `router/src/middleware/tracing.rs`
- `router/src/policy/account_constraints.rs`
- `docs/ops/router-fallback.md` (internal; minimal stub)

## Tests
- Unit tests: compat set evaluation for tool calling + JSON schema mode
- Integration test: pinned account with degraded primary route selects region fallback only

## Review notes
- @Priya Natarajan: dashboards can key off `router_fpe_activated_total{reason}` once metrics PR lands (separate).

## Conversation
Priya Natarajan: can we ensure data residency constraints are evaluated *before* reason code logic?
Ben Carter: yes, `account_constraints` runs pre-selection; reason code only indicates why primary was rejected.

Connor O'Brien: please add explicit error if config mistakenly provides variant fallbacks for pinned customers
Ben Carter: added warning at config load; we prune invalid fallbacks and emit `router.fpe_invalid_fallbacks_total`

Alex Martinez: make sure this matches ADR-0142 semantics
Ben Carter: aligned; added link in code comments

## Merge
- CI: pass
- Merged via squash
