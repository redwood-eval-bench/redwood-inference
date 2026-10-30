# kv-cache-segment-shrinker-and-soft-spill-controller

Motivation and summary: The KV cache in the serving runtime shows long-tail memory fragmentation during mixed-length workloads that leads to intermittent OOMs and higher-than-expected tail latency at high concurrency. This PR introduces two complementary runtime mechanisms: a conservative segment-shrinker to proactively reclaim small free regions without aggressive global compaction, and a soft-spill controller that prefers low-latency background spill to SSD/remote when memory pressure crosses a configurable soft threshold. Both mechanisms are opt-in behind a feature flag and are designed to reduce fragmentation and peak memory while preserving steady-state throughput.

What changed (high level):
- Implemented SegmentShrinker: periodically scans per-tenant segment lists and coalesces adjacent free ranges into larger free segments, avoiding full-spectrum compaction. Shrinker operates with a budgeted work quantum to bound CPU impact.
- Added SoftSpillController: when free memory falls below kv.soft_spill.threshold_percent, eligible cold segments are asynchronously spilled to configured spillback targets (local SSD or remote object store). Soft spill is non-blocking for the request path and uses a throttled worker pool.
- New telemetry: kv.segment_shrinks_total, kv.segment_shrink_cpu_ms, kv.soft_spill_bytes, kv.soft_spill_backpressure_count, kv.segment_free_bytes_histogram. These allow SRE to observe shrink/spill effectiveness and tune thresholds.
- Config + rollout: feature flag kv.segment_shrinker.enabled (default false) and kv.soft_spill.enabled (default false). Defaults make this a safe, gradual rollout for Dedicated and Private installs.
- Tests & benches: unit tests for shrinker invariants and integration smoke tests for spill-backpressure. Microbench harness added to benchmark fragmentation-reduction and 95th latency impact.

Commits (summary):
- 1/3: api/kv: add config knobs and feature flag plumbing, default off
- 2/3: runtime/kv: implement SegmentShrinker with budgeted worker and API to coalesce free ranges
- 3/3: runtime/kv: add SoftSpillController, telemetry, tests and bench harness

Files touched (high level):
- runtime/kv/shrinker.go (new): core shrinker logic and budgeting
- runtime/kv/soft_spill.go (new): spill controller and worker pool
- runtime/kv/allocator.go (mod): small allocator hooks to expose free-range iterators
- api/config/kv.yaml (mod): feature flags and thresholds
- docs/ops/kv-telemetry.md (new): operational guidance for metrics and thresholds
- tests/kv/shrinker_test.go (new), tests/kv/soft_spill_integration_test.go (new)
- bench/kv/fragmentation_bench.go (new): microbench harness

Bench & validation notes:
- Lab bench (N=10 runs, mixed short/long seq lengths) shows median memory fragmentation reduced by ~18% and 95th token latency improved by 3.8% under high concurrency (80th-100th percentile workload mix). End-to-end request throughput unchanged within noise.
- In staging under synthetic tenant load, observed one Mina-class OOM avoided during a 12-hour soak with kv.soft_spill.enabled=true and threshold 12% free.
- CPU overhead measured: segment shrinker set with a conservative budget of 5ms/second contributed ~0.7% CPU on average; knobs available to bump down to 1ms/s or disable entirely.

Review conversation (high level):
- Arjun: Requested more telemetry for tracing shrink decisions and a per-tenant throttle. Action: added kv.segment_shrink_decisions_total and per-tenant throttle via config with defaults.
- Liang: Asked for stronger invariants in allocator integration and an assert harness to catch races. Action: added assertions in allocator and extra unit tests that run under race build.
- Sofia: Concern about spill target reliability and backpressure path. Action: added backpressure counters and added defensive retry with exponential backoff for remote spills; documented failure modes in docs/ops/kv-telemetry.md.
- Author replies: addressed above comments in two follow-up commits; dropped CPU budget and added explicit feature flags.
- Final approvals: Arjun approved on 2026-02-23, Liang approved after small test refactor on 2026-02-24.

CI: All checks passed after one transient test rerun (kv/bench flaky on first run). CI summary: unit (pass), integration (pass), bench (pass after rerun), lint (pass), go vet (pass).

Deployment and rollback plan: Merged behind feature flags. Recommended rollout sequence:
1) Enable kv.segment_shrinker.enabled=true on a single low-traffic tenant for 24h.
2) Monitor kv.segment_free_bytes_histogram and kv.segment_shrinks_total; if shrink CPU > configured budget, reduce kv.segment_shrinker.budget_ms_per_sec.
3) Enable kv.soft_spill.enabled=true at low threshold (e.g., 8-12%) for limited tenants; monitor kv.soft_spill_bytes and kv.soft_spill_backpressure_count.
4) If any regression observed, flip flags off and collect artifacts (heap+profiles) for followup.

Release notes (for changelog):
- Added an opt-in segment shrinker and soft spill controller to reduce KV cache fragmentation and mitigate OOMs. Disabled by default; enable via kv.segment_shrinker.enabled and kv.soft_spill.enabled. Metrics and operational guidance added.

Risk and mitigations:
- Risk: misconfigured thresholds could increase remote egress costs (soft spill). Mitigation: default off and conservative thresholds + billing notice in docs.
- Risk: shrinker CPU interfering with hot-path. Mitigation: strict CPU budget and per-tenant config.

Follow-ups and TODOs:
- PERF-130: experiment with adaptive budget that scales with observed fragmentation rate.
- ENG-4825: add an automated rollout job to progressively flip feature flags based on telemetry.

Merge outcome: Squash-merged to main after approvals and green CI (merged_at 2026-02-24).
