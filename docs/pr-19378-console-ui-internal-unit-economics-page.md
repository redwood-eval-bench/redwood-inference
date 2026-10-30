# Console: add internal Unit Economics dashboard page (filters, charts, top‑N tables)

## Summary
Implements the internal Unit Economics dashboard page in Console and wires it to the backend query endpoints from PR #19374 (time series + breakdown). This is an internal-only page intended for Finance/Product/Eng to view and diagnose blended $/1M output tokens with drilldowns by model, region, and customer tier.

## UI shipped in this PR
- New route: `/internal/unit-economics`
- Filter bar:
  - Time range (Last 24h / 7d / 30d / Custom)
  - Group by (Model / Region / Customer tier)
  - Model selector (supports model + version label when present)
  - Region selector (canonical region values)
  - Tier selector (self-serve / dedicated / private / enterprise)
  - Comparison toggle (previous period)
- Primary charts:
  - Blended cost: `$ per 1M output tokens` time series
  - Output tokens time series (for context + data quality sanity checks)
  - Total allocated cost time series (optional toggle, default hidden)
- Breakdown section:
  - Top-N table (default N=10): shows blended $/1M output tokens, output tokens, allocated cost, cache hit rate (if present), batching efficiency (if present)
  - Sorting by blended cost desc (default) with ability to sort on tokens/cost
  - Row click drilldown: navigates to same page with prefilled filters (e.g., click a model row to lock model and switch group_by)
- Empty/error states:
  - Friendly empty state when the selected slice has no data
  - Error boundary + retry when APIs return non-200 or time out
  - “Data may be incomplete” banner when API sets `coverage_status != ok` (plumbed through, UI copy per DES-319)

## Access / internal-only behavior
- Page is behind an internal feature flag: `unit_econ_dashboard_internal`.
- Navigation item only appears when the feature flag is enabled.
- The page still relies on backend RBAC enforcement (API returns 403 if user isn’t allowed); UI shows a permission-denied panel with a link to request access.

## API wiring
- Uses the new Console query client wrappers:
  - `GET /console/internal/unit-economics/timeseries`
  - `GET /console/internal/unit-economics/breakdown`
- Requests are made with stable query keys so the results are cached in the client (React Query) and deduped when users quickly change filters.
- URL query params are the source of truth for filter state to enable shareable links (e.g., Finance can paste a link for “EU-West / dedicated / last 7d”).

## Implementation notes
- Added a small normalization layer on the client to keep the UI resilient:
  - Unknown/NULL dimension values are displayed as `Unknown` but retained as a selectable filter value.
  - Currency formatting per DES-319: $/1M output tokens shown with 2 decimals for values >= 1.00 and 3 decimals for values < 1.00.
  - Time zone: charts render in user local time, but we show a tooltip note “Aggregations are computed in UTC hourly buckets” to avoid day-boundary confusion.
- Kept the table columns flexible to support v1.1 additions without a rewrite (prefill/decode ratio, streaming flag, etc.).

## Screens / manual QA checklist
- [x] Flag off: page and nav do not render
- [x] Flag on + allowed user: charts/table render
- [x] Flag on + disallowed user: permission denied panel
- [x] Last 24h / 7d / 30d presets work
- [x] Custom range clamps to max 90 days (matches backend constraints)
- [x] Drilldown navigation preserves time range + comparison toggle
- [x] Sorting and pagination (Top-N) behave as expected

## Changed areas (high level)
- `console/web/src/routes/internalUnitEconomics.tsx` (new route)
- `console/web/src/pages/InternalUnitEconomicsPage/*` (page + components)
- `console/web/src/components/filters/UnitEconFilterBar.tsx` (new)
- `console/web/src/components/charts/CostPer1MOutputTokensChart.tsx` (new)
- `console/web/src/components/tables/UnitEconTopNTable.tsx` (new)
- `console/web/src/api/unitEconomics.ts` (query client wrappers)
- `console/web/src/nav/internalNav.ts` (conditional nav item)

## Commits
1) Add internal route + skeleton page
2) Implement filter state in URL + defaults
3) Wire time series endpoint + chart rendering
4) Wire breakdown endpoint + Top-N table + drilldown behavior
5) Add empty/error states, permission denied view
6) Polish formatting + design alignment (DES-319)

## Follow-ups (not in this PR)
- Saved views (Finance presets) and “live vs finance view” toggle (discussed in Slack thread on caching vs freshness)
- Add confidence indicator calculation details once finalized in the metrics layer
- Add additional columns when telemetry signals land consistently (batching utilization, prefill/decode ratio)

---

### Review conversation (abridged)
**Alicia Perez (Design)**: The cost chart y-axis label should be explicit about units. Also please avoid showing too many decimals; it reads noisy.

**Nora Ahmed (Author)**: Updated label to “Blended cost ($/1M output tokens)” and applied DES-319 rounding rules (2 decimals >= 1, 3 decimals < 1). Pushed in 9d2a8c1.

**Felix Schneider**: Can we make URL params the source of truth? Right now some local state can drift when using browser back/forward.

**Nora Ahmed (Author)**: Good catch. Refactored to derive all filter state from `useSearchParams` and only write via `setSearchParams`. Back/forward works now. See 41bf02a.

**Chloe Martin (Backend)**: The breakdown endpoint can return `unknown` for region/tier; please don’t drop those rows. Also the API caps `top_n` at 50.

**Nora Ahmed (Author)**: UI now shows `Unknown` as a value and keeps it in the table; `top_n` selector clamps at 50. Added a small note in the empty state when all rows are Unknown (often indicates mapping issues).

**Allison Grant (Observability)**: Please add a basic client-side guard so we don’t spam the API on every keypress in the model filter.

**Nora Ahmed (Author)**: Added debounce (250ms) on searchable selects and ensured React Query keys are stable. Also added a request cancellation on rapid changes.

### CI
- Initial run failed due to a snapshot mismatch for the filter bar (expected after label tweak). Updated snapshots.
- Final: web-build, typecheck, lint, unit tests all green.

### Outcome
Merged to `main` (squash). Internal dogfood announcement will go out after RBAC/feature-flag allowlist ships (PR #19397).
Internal-only Console page for Unit Economics dashboard (blended $/1M output tokens) with filters + drilldowns wired to unit economics query APIs.
