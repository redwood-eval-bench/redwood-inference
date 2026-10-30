# Add internal admin CLI for scaling policy debug (effective policy + decisions + bound-hit history)

Context: During Dedicated autoscaling bounds + predictive mode pilot, on-call and CS have needed a fast way to answer: (1) what is the *effective* scaling policy for a deployment (after defaults + feature flags), (2) what has the autoscaler been deciding recently (requested desired vs bounded desired + reason), and (3) are we frequently hitting min/max bounds. This PR adds an internal-only admin CLI command to dump that info without spelunking multiple dashboards/log streams.

What’s in this PR: - New `rw-admin` subcommand: `rw-admin scaling policy debug` - Outputs:   - Effective scaling policy (raw config + evaluated defaults + feature-flag overrides)   - Last N scaling decisions (timestamp, requested_desired, bounded_desired, clamp_reason, signals summary)   - Bound-hit history summary (last 1h/6h/24h counters + max-bound saturation ratio) - Supports `--format=table|json` (json is intended for copy/paste into incident docs / support tickets) - Supports `--deployment-id` (required) and optional `--region` (defaults to deployment’s primary region) - Supports `--since` (e.g. `2h`, `24h`) and `--limit` for decisions - Adds guardrails: admin-only auth token required; redacts customer name by default; no prompt contents or request IDs printed.

Implementation notes: - CLI is Go (cobra) under `cmd/rw-admin` (internal toolchain already used by SRE). - Uses capacity-policy-service internal endpoints added in prior work:   - `GET /internal/v1/deployments/{deployment_id}/scaling/effective-policy`   - `GET /internal/v1/deployments/{deployment_id}/scaling/decisions?limit=...` - Bound-hit history is fetched via metrics-proxy (Prometheus query) to avoid direct access to Prometheus from laptops. The query uses low-cardinality labels (deployment_id, region) consistent with observability standards.

Example output (table):

$ rw-admin scaling policy debug --deployment-id dep_7f3a2 --since 6h
Effective policy:
  enabled: true
  min_replicas: 6
  max_replicas: 24
  cooldown_up: 60s
  cooldown_down: 180s
  predictive_mode: enabled (fallback=reactive)
  clamp: hard (min/max enforced)

Recent decisions (last 10):
  2025-08-13T00:41:12Z requested=18 bounded=18 reason=none signals=qdepth_high
  2025-08-13T00:39:12Z requested=26 bounded=24 reason=max_bound signals=predicted_ramp
  2025-08-13T00:37:12Z requested=26 bounded=24 reason=max_bound signals=predicted_ramp

Bounds summary (since 6h):
  max_bound_hit_count: 42
  min_bound_hit_count: 0
  max_bound_saturation_ratio: 0.18

Example output (json):

$ rw-admin scaling policy debug --deployment-id dep_7f3a2 --since 24h --format json | jq
{
  "deployment_id": "dep_7f3a2",
  "region": "us-east-1",
  "effective_policy": { ... },
  "recent_decisions": [ ... ],
  "bound_hit_summary": { ... }
}

Testing: - Unit tests for command parsing and response rendering - Golden tests for `--format=table` output stability - Smoke test for metrics-proxy query construction (no live Prometheus dependency)

Rollout/usage: - Internal-only binary; no customer-facing surface - Intended for SRE + Dedicated on-call + CS escalations (pairs with runbook section “Confirm effective policy + bound hits”).

Follow-ups (not in this PR): - Add `rw-admin scaling policy validate` to preflight proposed min/max bounds against current fleet availability by region (tracked separately).
Ethan Park (review): Can we confirm these endpoints are strictly internal and require admin auth? Also please ensure the CLI doesn’t print customer names by default (compliance). Jordan Lee (author): Yep — using the same admin token flow as other rw-admin commands; added `redactCustomerName()` with an explicit `--include-customer` flag (off by default).

Vanessa Ortiz (review): For “recent decisions”, can you include both requested_desired and bounded_desired plus the clamp reason. That’s the most useful part when diagnosing max bound saturation vs predictor overshoot. Jordan Lee: Added `requested_desired`, `bounded_desired`, and `clamp_reason` columns plus a compact signals string.

Allison Grant (review): Prometheus query should not use customer_id label (cardinality). Please query by deployment_id + region only. Jordan Lee: Updated metrics-proxy queries to use deployment_id + region; removed optional customer label entirely.

Rafael Mendes (review): Please add a short note to the Dedicated on-call docs with a copy/paste command; also make sure error messages are actionable when the metrics-proxy is unreachable. Jordan Lee: Added docs snippet in `docs/internal/oncall/rw-admin.md` and improved errors to suggest `--format json` and to check VPN/SSO.

Felix Schneider (review): JSON output should be stable (field names) since we paste into incident timelines. Also prefer RFC3339 timestamps. Jordan Lee: All timestamps are RFC3339 and JSON fields are fixed; added a small golden test for json keys order stability (via canonical encoder).

Outcome: Approved by all reviewers; merged squash.
Internal tooling only: new `rw-admin scaling policy debug` command to dump effective scaling policy, recent autoscaler decisions, and bound-hit history for a Dedicated deployment.
