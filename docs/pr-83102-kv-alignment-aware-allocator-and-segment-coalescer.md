# Alignment-aware allocator and segment coalescer for KV cache

Motivation: The KV cache has accumulated significant internal fragmentation on long-running hosts serving heterogeneous session lengths. Fragmentation increases memory footprint, triggers more frequent spillover/paging, and amplifies eviction churn under load. This PR introduces an alignment-aware allocation strategy plus a background segment coalescer and a threshold autotuner to reduce fragmentation and smooth spillover behaviour.

Summary of changes: 
- New alignment-aware slab allocator: buckets allocations to power-of-two alignment classes chosen to reduce wasted space for common KV sizes (8B, 32B, 128B, 512B, 2KB). The allocator records an "aligned_alloc_ratio" metric.
- Segment coalescer: background worker that merges adjacent free segments within a shard when fragmentation ratio exceeds a dynamic threshold. Coalescing is cooperative and throttled to avoid impacting serving latency (hot-path is lockless; coalescer uses try-lock ramps).
- Threshold autotuner: samples fragmentation, page-fault/spill events, and batch-insert rates to tune coalescing aggressiveness and per-bucket split thresholds at runtime. Tunable via feature flag and config knobs.
- Soft backpressure hook for continuous batching: large insert bursts now see soft delays coordinated with the allocator to avoid creating many tiny free segments that trigger immediate spillover under pressure. This integrates with the existing batching controller to apply token-level pacing.
- Telemetry & CI: new metrics and dashboard snippets; unit and stress tests added.

Why this is different from prior work: Prior PRs focused on eviction policies and spillback bisection. This change attacks the root cause: internal fragmentation and allocation alignment, reducing the amount of memory that must be reclaimed and therefore reducing eviction churn and paging.

Checklist: 
- [x] allocator implementation and unit tests
- [x] background coalescer + integration tests (simulated shard stress harness)
- [x] autotuner with conservative defaults and opt-out
- [x] metrics and console dashboard wires
- [x] canary rollout & feature flagging

Operational notes: The feature is gated behind the runtime flag "kv.align_coalesce_enabled" (default: false). We will enable it for perf-canary nodes for 48h, then a 10% production canary before full rollout. If regressions are observed, toggle the flag off — changes are additive and safe to revert at runtime.

Files changed count and size are moderate; this touches allocator, shard bookkeeping, and observability layers. Details in commits below.
1) allocator: add alignment-aware buckets and allocation helpers (meera/kv/allocator)
2) coalescer: background worker and heuristics + integration with shard free list
3) autotuner: sampling/decision loop and config binding
4) batching: soft backpressure hook and tests
5) metrics: add kv.aligned_alloc_ratio, kv.coalesce_ops, kv.soft_bp_events; dashboards
6) tests & docs: unit + stress tests, rollout docs and feature flagging
Sanjay Patel (2026-02-19): Nice direction. Two questions: what's the expected CPU overhead of the coalescer under steady state? And can the autotuner be bounded to avoid oscillation?
Meera Kline (2026-02-19): Coalescer is designed to be low-priority; in microbenchmarks we saw ~+2% CPU on median but no impact on p99 latency. Autotuner uses hysteresis and a dampening window (configurable) to avoid oscillation — added notes in doc.
Alex Morgan (2026-02-20): Tests look good; add a deterministic seed for the stress harness so flakiness is debuggable. Also please ensure metrics are tagged by shard and region.
Meera Kline (2026-02-20): Added deterministic seed to tests and per-shard tags to metrics. Updated tests and dashboard wiring.
Priya Desai (2026-02-22): Approve after CI rerun and a short perf summary in the PR body. Please include rollout steps in runbook.
Meera Kline (2026-02-22): Added perf summary and runbook section. Marking ready for merge.
Implements an alignment-aware allocator and background segment coalescer for the KV cache; reduces internal fragmentation and spill-triggered evictions. Feature gated behind kv.align_coalesce_enabled. See runbook for canary rollout.
Bench summary (perf-canary sim, 64-concurrency nginx-like workload):
- memory footprint: -9% (small-session heavy) to -14% (mixed workloads)
- eviction churn: -15% overall
- p50 token latency: ~0% change
- p99 token latency under heavy insert bursts: -8%
- CPU median: +2% (coalescer), peak unchanged
Notes: results vary by model and session distribution; autotuner defaults are conservative to prioritize p99 latency.
1) Enable kv.align_coalesce_enabled on perf-canary nodes (48h)
2) 10% production canary (72h) monitoring kv.frag_ratio, kv.soft_bp_events, p99 latency, and OOM/eviction rates
3) Gradual ramp to 100% if metrics stable; revert flag immediately if regressions observed
4) Post-rollout: ramp up coalescer aggressiveness via autotuner if frag_ratio stays > 0.25 for >24h
This PR intentionally keeps the hot-path allocation fast and lockless; heavy work is offloaded to the coalescer. The autotuner can be disabled by setting kv.align_coalesce_autotune=false for customers requiring deterministic allocation behaviour.
