# Usage insights: add data freshness metadata + console banner for partial/lagging data

Context: during beta testing we’ve had multiple instances where customers interpreted route/model breakdown swings as “real spend changes” when the underlying issue was ingestion lag / late events / partial rollups for a region. This PR adds explicit freshness metadata to the Usage Insights API and renders a banner + inline status on the console breakdown pages when results may be incomplete. The goal is to make data trust issues obvious, reduce support load, and give users an action (narrow time range / wait for backfill) rather than silently showing partial totals.

What this PR does:
- API: extends v1 usage-insights responses to include `data_freshness` metadata (latest_complete_bucket_end, observed_lag_seconds, completeness, and a human-readable status reason). This metadata is computed per query based on the requested time range and the dataset watermark.
- Console: shows a dismissible banner at the top of Usage → Cost Insights pages when `completeness != complete` or when lag exceeds threshold; adds a smaller “Data freshness” pill next to the time range selector for non-blocking warnings.
- UX details: banner copy is intentionally specific (“Data through 14:00 UTC; newer data still processing”) and links to the help article section on freshness/late events.
- Telemetry: emits a `console.usage_insights.freshness_banner_shown` event with the reason code so we can measure how often users hit lag/partial states.

Thresholds / semantics:
- `completeness` is `complete` if requested_end <= latest_complete_bucket_end.
- `completeness` is `partial` if requested_end > latest_complete_bucket_end.
- `completeness` is `unknown` if the watermark lookup fails (we default to no banner, but show a small pill tooltip indicating we couldn’t confirm freshness).
- Banner is shown for `partial` always; for `complete` we show only if observed_lag_seconds > 900 (15m) to catch edge cases where buckets are “complete” but significantly delayed.

API example (abridged):
- `data_freshness.completeness`: complete | partial | unknown
- `data_freshness.latest_complete_bucket_end`: RFC3339 timestamp
- `data_freshness.observed_lag_seconds`: integer
- `data_freshness.reason`: enum (LATE_EVENTS, AGGREGATION_LAG, WATERMARK_UNAVAILABLE)

Notes:
- This is backwards compatible for existing clients; the new field is additive.
- The freshness metadata is explicitly excluded from response cache keys (it’s computed post-query from the same watermark source), but the watermark lookup itself is cached for 30s in the query service to avoid thundering herds.

Testing:
- Unit tests for freshness computation across time ranges + watermark states.
- Frontend tests for banner rendering/copy and dismissal persistence.
- Manual: verified in staging by forcing a low watermark and querying “Last 24h”.

Rollout:
- Guarded behind existing usage-insights beta flag; banner can be disabled via `console_usage_insights_freshness_banner` kill switch if copy/threshold needs rapid iteration.

Refs:
- Linear: ENG-3824
- Related: PR #23135 (query API v1), PR #23114 (aggregation dataset)
- Support signal: SUP-24863 (timeouts/long ranges often correlating with partial data); this should reduce confusion when users narrow ranges.

Checklist:
- [x] Add API contract + changelog entry for additive field
- [x] Console banner + pill states (partial/lag/unknown)
- [x] Events + dashboards hook (follow-up: add alert if watermark unavailable > 5m)
- [x] Copy reviewed by Design + DevEx (lightweight) and aligned with help center wording

Chloe Martin (requested changes): Can we make sure the watermark lookup failure doesn’t flip the whole request to 500? I’d prefer the query still succeeds and freshness is `unknown`.
Felix Schneider (author): Yep — updated to catch watermark errors and set `completeness=unknown` + `reason=WATERMARK_UNAVAILABLE`. Query response remains 200 and we don’t show the big banner by default (only pill tooltip). Added test coverage.
Allison Grant: The reason codes look good, but we should be careful about calling it “late events” vs “aggregation lag”. Can we ensure `LATE_EVENTS` is only used when the late-event counter is non-zero for the window?
Felix Schneider (author): Good call. I split it: if `late_event_rate > 0.5%` for the trailing hour, reason is `LATE_EVENTS`; otherwise `AGGREGATION_LAG`. If we can’t fetch the metric, we default to `AGGREGATION_LAG` to avoid over-claiming. Tests updated.
Tessa Morgan (requested changes): Banner copy feels too generic. Can we include a concrete “Data through <time>” and avoid “may be incomplete” if it definitely is incomplete?
Felix Schneider (author): Updated copy to: “Showing partial data. Complete data through {timestamp} (UTC). Newer usage is still processing.” and only uses “may” for `unknown` (pill tooltip).
Kaitlyn Nguyen: RBAC question — are we leaking anything about other projects/workspaces via watermark? This should be scoped to workspace/project filter.
Felix Schneider (author): Watermark query is computed on the same dataset partition keys as the request (workspace_id + project_id when provided). No cross-scope access. Added an auth regression test to ensure watermark lookup uses the caller’s scoped filter set.
Dylan Brooks: Looks solid. One perf nit: avoid an extra warehouse roundtrip for watermark when the request is cached.
Felix Schneider (author): Implemented 30s in-process cache for watermark keyed by (workspace, project, dataset, bucket). Also ensured we compute freshness after retrieving cached breakdown results so we don’t invalidate existing caches.
Allison Grant (approval): Approved after changes. Please make sure we track watermark-unavailable rate post-merge.
Dylan Brooks (approval): Approved.
