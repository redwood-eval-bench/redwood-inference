# Console: Usage Insights docs (route/model breakdown, cost drivers, exports, troubleshooting)

## Summary
Adds the initial help-center / docs-site page for **In-console Usage Insights (Beta)**. The doc explains how the route+model breakdown is computed, defines the key metrics (tokens in/out, cache, retries/fallbacks, streaming overhead), and includes a troubleshooting section based on early beta/support feedback. This is intended to ship with the Console beta rollout and will be linked from the Usage Insights pages.

## What’s included
- New doc page: **Console → Usage → Usage Insights (Beta)**
- Definitions and attribution semantics (aligned with platform ADR/API contract)
- Guidance on interpreting cost drivers and “What changed?” deltas
- Export (CSV) overview + permissions/audit expectations
- Troubleshooting / known issues (billing reconciliation, missing route tags, time range timeouts)
- Screenshot placeholders with callouts (to be replaced once final UI polish lands)

## Notes on terminology
We use **Route** in the Console UI and docs to mean the canonicalized request path + method as recorded by the gateway (e.g., `POST /v1/chat/completions`). Where customers expect “endpoint,” we call out that Route ≈ Endpoint and explain normalization (path params templated).

## Security/RBAC
Docs explicitly call out that cost visibility and exporting are gated by workspace settings and role permissions. Also notes that export/download events are audit-logged (field set referenced in INT-9140).

## Screenshot placeholders
Added `TODO(screenshots)` blocks with suggested capture specs:
- Workspace selector + time range
- Breakdown table (Route × Model) with sorting
- Drilldown view with context length histogram and cache hit rate
- Top Cost Drivers card(s)
- Export flow modal + permission denied state

## Related work
- Console UI: redwood/pr-23141
- Export endpoints + signed downloads: redwood/pr-23148
- RBAC gating: redwood/pr-23160
- Data freshness banner: redwood/pr-23170

## Checklist
- [x] Align metric definitions with Console tooltips (DevEx feedback)
- [x] Include billing reconciliation caveats and expected tolerances
- [x] Add troubleshooting entries for known beta issues (SUP-24811 / SUP-24827 / SUP-24863)
- [x] Security review callout for audit logs + export permissions
- [x] Mark as Beta + document limitations

## Open items (follow-ups)
- Replace placeholders with final screenshots after UI copy freeze (target: before beta announcement post)
- Add a short “How to export via API” snippet once examples repo script is public (redwood-examples/pr-622)
- Expand section on model aliases/version bucketing once ENG-3823 fully lands (avoid duplicate-row confusion)
Elliot Price (review): Can we add a simple, explicit definition of Route (method + canonical path) right near the top? Also please include a note about time zone handling (Console uses workspace time zone; exports are UTC).
Wei Chen (author): Yep. Added a "What is a Route?" section above the first table and a short time zone note in Filters + Export sections.
Alex Martinez (review): Good overall. Two nits from DevEx feedback: 1) Avoid calling it "endpoint grouping"—just say route normalization. 2) Add a one-liner that Route ≈ Endpoint for people coming from other dashboards.
Wei Chen (author): Updated phrasing + added the Route ≈ Endpoint callout under Terminology.
Marissa Cole (review): Please be careful on the "What changed?" section: we should not imply causality. Can we phrase drivers as "correlates" and make it clear it’s heuristic-based? Also add a small note that "Top cost drivers" may be limited when data is partial/lagging.
Wei Chen (author): Adjusted copy to "signals" / "correlates" and added a partial-data caveat + link to freshness banner explanation.
Marcus Lin (review - security): Export section needs a stronger statement about permissions + audit logging. Mention that download links are signed and time-limited; include that exports may be disabled for restricted workspaces. Also link INT-9140 requirements (without exposing internals).
Wei Chen (author): Done: added "Security and permissions" subsection, included signed URL TTL language (non-specific), and noted restricted workspace behavior. Kept it customer-facing (no internal ticket details).
Dylan Brooks (review): Can we add a troubleshooting entry for "Insights page times out on 30d" with recommended mitigation (shorter ranges) and explanation that perf is improving?
Wei Chen (author): Added a Q/A referencing SUP-24863, plus a note about long ranges and exports as an alternative.
Elliot Price (approval): Approved with minor copy tweaks. Please ensure all TODO(screenshots) are consistent and searchable.
Wei Chen (author): Standardized TODO markers and added a small checklist in the doc header for screenshot replacement.
