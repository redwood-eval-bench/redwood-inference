# kv-slot-batcher: reserve adapter for lightweight slot batching and prefetch steering

Problem: Small, concurrent KV cache reservations for short-context workloads generate many tiny allocations and scheduler handoffs, which increases per-request CPU overhead and creates noisy eviction churn. On high-concurrency generation workloads (many short prompts with streaming), we observed ~18% p50 latency regression and higher tail jitter due to reservation contention and immediate prefetches for marginally useful slots.

Goals: Reduce per-reservation overhead, aggregate small reservations to amortize synchronization, and provide lightweight steering hints to the prefetcher so it focuses on likely hot slots. All changes must be backward compatible with existing slot ownership semantics and respect existing pin/eviction guarantees.

What this PR does:
- Introduces a ReserveAdapter abstraction that sits between caller code (kv_cache::reserve_path) and the underlying block/slot allocator. The adapter performs short-lived aggregation of reservation requests into SlotBatches, holding reservations in a cheap local pool for O(1-5ms) to improve reuse and coalescing.
- Adds a SlotBatcher implementation (slot_batcher.rs) that groups reservations by affinity key (model+request bucket) and sequence length bucket to maximize locality. Batches are size-capped and flushed on a combination of time and backpressure thresholds.
- Adds prefetch steering hooks: the adapter emits lightweight, best-effort steering hints to the prefetcher (prefetcher.rs) so the prefetcher prioritizes batches with higher reuse probability. Hints are advisory; no correctness depends on them.
- Soft-lease counters: rather than hard-pinning on reservation, we introduce a soft-lease counter that allows the eviction fastpath to be aware of recent adapter activity and avoid immediately reclaiming freshly-batched slots. This avoids eviction ping-pong on workloads with rapid reuse.
- Scheduler integration: hint_router.rs consumes adapter hints and maps them to scheduler priorities when routing requests to serving pools. This reduces head-of-line blocking for high-locality batches.
- Tests and microbench: added a microbench harness that simulates 1000 concurrent short prompts to validate latency and throughput improvements and new unit tests for the adapter lifecycle.

Design notes and tradeoffs:
- The ReserveAdapter is intentionally lightweight: it retains reservations only for a short dwell period to increase reuse without violating observable semantics. If a reservation is not reused within the dwell window, it's flushed to the allocator as before. This avoids long-lived implicit pins.
- Batching uses affinity keys to maintain strong locality; affinity keys are coarse (model + bucket) to avoid over-sharding.
- Steering hints are lossy and non-authoritative: they only influence prefetch ordering. If the prefetcher is disabled the system still functions identically to today.
- We intentionally avoid adding additional persistent metadata to the block header to keep on-disk compatibility intact. All adapter state is in-memory and ephemeral.

Benchmark results (local perf lab, p95 runs):
- Baseline: 1000 concurrent short-stream requests, p50 latency 18.4ms, throughput 9.4k tokens/s, CPU 78%
- With ReserveAdapter: p50 latency 15.1ms (-18%), throughput 10.8k tokens/s (+15%), CPU 66% (-15% CPU), tail jitter reduced (p99 / p50 ratio improved ~22%)
- Memory: transient adapter overhead ~1.5MB per serving worker under test config; configurable via adapter.max_memory_bytes.

Checklist (both checked in code where applicable):
- [x] Unit tests for adapter lifecycle and batch flush semantics
- [x] Microbench harness reproducing high-concurrency short-context workload
- [x] CI integration: lint, unit, integration microbench step
- [x] Config knobs and defaults exposed via runtime config with safe conservative defaults
- [x] Design doc: docs/design/kv-slot-batcher.md

Files changed (high-level):
- src/runtime/kv_cache/reserve_adapter.rs: new core. Contains ReserveAdapter, lightweight timing wheel, and metrics emission.
- src/runtime/block_manager/slot_batcher.rs: new batching policy, capacity accounting, and flush logic.
- src/runtime/prefetch/prefetcher.rs: accept optional advisory steering hints; small reordering logic.
- src/runtime/scheduler/hint_router.rs: minor router extension to map hints to priority lanes.
- src/runtime/kv_cache/mod.rs: wiring into existing reserve_path entrypoint and config knobs.
- tests/microbench/kv_slot_batcher.rs: harness and reproducible scenarios.
- docs/design/kv-slot-batcher.md: rationale, knobs, and perf numbers.
- Cargo.toml: added small dev-dep for bench harness.

Review conversation (high-level chronology):
- Ravi: "Nice clean separation. Two requests: (1) can we make the dwell window configurable per-pool? (2) please add a metric for 'batched_reserve_hit_rate' so we can SLO on expected reuse."
  - Maya: "Added pool-level dwell window (config + safe default) and a new metric 'kv.reserve_adapter.batch_hit_ratio' exposed via metrics crate. See last commit e4a1d07."
- Sofia: "Concern about eviction interaction: if batches keep slots alive, will this starve other tenants?"
  - Maya: "Adapter uses soft-lease counters with max_soft_lease_ms; eviction code checks combined hard pins + soft_lease and falls back to reclamation under memory pressure. I added comments and a unit test validating reclamation under forced pressure."
- Liam: "Microbench was convincing. Can we add a short integration test that runs the hint path through the scheduler to ensure no regressions?"
  - Maya: "Added a scheduler routing smoke test in microbench. It exercises hint->router->pool path."
- Ravi (approve): "LGTM after docs update"
- Sofia (approve): "Verified on perf lab, merging after CI passes"

CI: All checks passed (lint, unit, integration microbench using perf-canary harness). No flaky runs in the observed CI window.

Follow-ups and future work:
- Explore adaptive dwell window based on measured reuse heat (ENG-2334).
- Consider exposing per-model affinity tuning via config center for large customers.
- Longer-term: merge steering hints with predictive prefetch models in prefetcher v2.

Notes: This change targets runtime performance and is safe for release without customer-facing API changes. Ops: rollout plan is a staged flag rollout (off by default), enable for a subset of serving pools, validate metrics, then flip globally if stable.

Related resources: Linear ENG-2217 (kv reservation batching), ENG-2334 (prefetch steering followup). Design doc: docs/design/kv-slot-batcher.md (included in this PR).
Introduce a lightweight reserve adapter that batches small KV slot reservations and supplies prefetch steering hints to the prefetcher. Improves median latency under high-concurrency workloads and reduces per-token CPU overhead. No API/compatibility changes.
