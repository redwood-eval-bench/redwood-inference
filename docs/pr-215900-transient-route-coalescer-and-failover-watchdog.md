# Introduce transient route coalescer and graceful failover watchdog

Motivation: During a multi-region load spike we observed repeated transient egress congestion that caused many in-flight requests to be re-routed, which magnified tail latency and triggered unnecessary regional failovers. This change introduces a lightweight transient route coalescer plus a graceful failover watchdog to reduce noisy re-routing and make SLO-driven failovers more deterministic.

What this PR does: - Adds a route coalescer that groups short-lived egress bursts within a tunable window so that routing decisions are evaluated on aggregated signals rather than per-request jitter. - Adds a failover watchdog that enforces a backoff+holdoff policy before initiating regional failover, driven by calibrated SLO breach heuristics. - Introduces telemetry hooks and a small in-memory smoothing buffer for egress metrics used by the coalescer. - Adds a new SRE-configurable feature flag (runtime) and a safe rollout plan (gradual enablement across regions).

Behavioral summary: - Short transients (default window: 500ms) are coalesced: routing retains the existing path while metrics accumulate; only sustained degradation triggers a failover candidate evaluation. - The watchdog requires a two-step confirmation (sustained metric, and KV-cache pressure or queue depth threshold) before promoting a failover. - Early exit on obvious catastrophic signals (node offline, disk threshold) remains immediate.

Testing and validation: - Unit tests for coalescer logic and smoothing buffer with edge cases (burst only, long sustained spike, interleaved quiet/burst). - Integration test simulating regional egress spike with synthetic traffic to validate reduced failover rate and improved P99 latency. - Load test in perf-lab: when coalescer enabled, failover decisions dropped by ~62% on short spikes and application P99 improved by ~18ms under same load profile.

Rollout plan: - Feature flag default off. Stage 1: enable in staging and run perf harness for 48h (done). Stage 2: enable in canary region (eu-west-1) with 1% of traffic for 6h -> 10% for 12h -> 100% for region. Stage 3: enable global by region when canary metrics stable. SLO guardrails in console will auto-roll back if region-level error budget drains > 4% within an hour.

Operational notes: - New metrics: egress_coalescer.windowed_events, failover_watchdog.confirmations, route_coalesce_skipped_reasons. - Two new alert rules added: 'coalescer_rate_anomaly' and 'watchdog_confirmation_spike'. - Recommended tuning: window_ms, min_events_to_coalesce, confirmation_threshold.

Migration/backwards compatibility: - All changes are opt-in via runtime config. No data migrations required. Fallthrough logic ensures behavior is identical to current routing when the feature is disabled.

References: Linear ENG-4219 for design doc, SRE runbook SRE-88 updated with tuning guidance.
Maya Chen: Can you add a short note about worst-case added latency from the smoothing buffer? We need to ensure targets are clear for mobile customers.
Author (Ethan Park): Added comments and a microbenchmark section in docs that shows median added latency < 0.6ms for default window. Updated docs/sre/runbooks/egress-coalescer.md.
Priya Nair: The confirmation thresholds look conservative; please add toggles so SRE can adjust without deploy. Also please ensure metrics are high-cardinality safe.
Author (Ethan Park): Exposed min_events_to_coalesce and confirmation_threshold to runtime flags; metrics tagged at region+route but aggregated in low-cardinality roll-ups. Added tests for tag merging.
Liam O'Reilly: Tested in perf-lab — behavior looks stable. Only request: add a short circuit if the KV-cache pressure cannot be read (GRACEFUL fallback).
Author (Ethan Park): Implemented KV-cache read guard with conservative fallback (treat as negative confirmation). Added unit test.
