# Monitor adapter: query latency + error SLO burn-rates (with staleness + sample sizes)

Context: Auto-rollback on SLO breach needs the rollout controller to fetch burn-rate status for the exact rollout scope (route/model_version/region/customer_tier) and make decisions with guardrails (sample size + staleness handling). This PR adds a monitor adapter in `redwood` that queries the internal slo-toolkit and normalizes results for the rollout controller.

What’s in this PR:
- New `SloMonitorAdapter` interface + `SloToolkitAdapter` implementation.
- Fetches burn-rate status for latency (p95/p99) and error-rate SLOs, returning both fast/slow windows (e.g., 5m/30m) and per-window sample sizes.
- Staleness detection:
  - Treats a monitor result as STALE when `now - data_timestamp > staleness_threshold` (default 2m; configurable).
  - Treats missing timestamps or NaN burn-rate as UNKNOWN (distinct from OK/BREACH).
- Resilience:
  - Timeouts + bounded retries when calling slo-toolkit.
  - Small in-process cache (TTL 10s) keyed by (scope + slo_type + objective) to avoid hammering slo-toolkit during rollouts.
  - Explicit UNAVAILABLE state when toolkit is down; includes error string for auditability.
- Adds adapter-level metrics (low-cardinality) for call volume/errors/staleness/latency.
- Tests: unit tests for staleness + parsing + caching behavior; integration test with a fake slo-toolkit server.

Notes / follow-ups:
- The rollout controller will still apply its own minimum-sample gating + debounce (ENG-4861 / PR-28463). This adapter returns sample sizes and timestamps so the controller can decide fail-open vs fail-closed per policy.
- slo-toolkit endpoint used here expects rollout-scoped labels; the corresponding toolkit work is in slo-toolkit PR-221.

Screenshots: N/A (backend-only).

Risk: Moderate. Adapter is behind `rollouts.slo_adapter.enabled` and is not invoked unless the rollout policy enables SLO-based actions.

Rollout plan:
- Merge behind flag.
- Enable in staging for internal canary route `chat/completions` (internal tenant only).
- Observe adapter error rate + staleness metrics for 48h before expanding.

Checklist:
- [x] Unit tests
- [x] Integration test with fake toolkit
- [x] Metrics added (cardinality reviewed)
- [x] Flag default OFF
- [x] Docs/commented example response payload
TypeScript (normalized response):

export type SloEvalState = 'OK' | 'BREACH' | 'UNKNOWN' | 'STALE' | 'UNAVAILABLE';

export interface WindowBurnRate {
  window: '5m' | '30m';
  burnRate?: number; // undefined when UNKNOWN/UNAVAILABLE
  sampleSize?: number; // requests in window (or undefined when unknown)
}

export interface SloMonitorSnapshot {
  scopeKey: string; // stable hashed key (no high-cardinality raw labels)
  sloType: 'latency_p95' | 'latency_p99' | 'error_rate';
  objective: string; // e.g. "p95<=1200ms" or "error_rate<=0.5%"
  state: SloEvalState;
  fast: WindowBurnRate;
  slow: WindowBurnRate;
  dataTimestamp?: string; // RFC3339 from provider
  fetchedAt: string; // RFC3339 now
  provider: 'slo-toolkit';
  providerMonitorId?: string;
  error?: string;
}

Sean Gallagher (SRE) - 2026-02-14: Can we make the staleness threshold explicit in config and ensure STALE is distinct from UNKNOWN? For oncall we need to tell the difference between "no data yet" and "provider is lagging".
Ethan Park (author) - 2026-02-14: Yep. Added `rollouts.slo_adapter.staleness_threshold` (default 2m) and split states: UNKNOWN (missing/invalid fields) vs STALE (timestamp too old). Updated tests.
Aisha Bello (Observability) - 2026-02-15: Please confirm metrics are low-cardinality. Avoid emitting raw route/model_version as labels here; those belong in audit events, not metrics.
Ethan Park (author) - 2026-02-15: Agree. Metrics only include `slo_type`, `result_state`, and `provider`. Scope is collapsed into a stable hashed key only in logs/audit payload (not metrics).
Jordan Lee (Policy) - 2026-02-15: The controller needs sample sizes per window. Right now you return `requests_total` only. Can you provide sample size for fast/slow windows separately?
Ethan Park (author) - 2026-02-16: Updated response model to include `fast.sampleSize` + `slow.sampleSize` and parse from toolkit payload. Added integration test fixture to validate both windows.
Benji Okafor (Policy/Config) - 2026-02-16: Minor: naming `objective` is a bit ambiguous—could be threshold string. Please add a comment and ensure it matches what we’ll store in the audit record.
Ethan Park (author) - 2026-02-16: Renamed internal var to `objectiveString` and kept serialized field as `objective` per ADR draft. Added comment + doc snippet.
Logan Wright (Telemetry) - 2026-02-17: Retry behavior: please cap retries to prevent thundering herd during provider brownouts. Also coalesce concurrent requests per key if possible.
Ethan Park (author) - 2026-02-17: Capped to 1 retry (2 total attempts) with jittered backoff; added simple in-flight promise map per cache key so concurrent calls coalesce. Tests added.
Sean Gallagher (SRE) - 2026-02-18: Looks good. One last ask: on UNAVAILABLE include a short error string but avoid leaking internal URLs. Approved after that.
Ethan Park (author) - 2026-02-18: Done—sanitize error messages to remove host/path and keep `code` + `message` only. ✅
Sean Gallagher (SRE) - 2026-02-18: Approved.
Aisha Bello (Observability) - 2026-02-18: Approved.
Jordan Lee (Policy) - 2026-02-18: Approved.
