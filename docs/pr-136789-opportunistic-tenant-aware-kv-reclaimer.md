# opportunistic-tenant-aware-kv-reclaimer-with-deferred-merge

Motivation: On mixed tenant workloads we observed increased KV-cache fragmentation and repeated small-object evictions during sustained tail-latency bursts. Existing slab and spill heuristics react too late (full-page spill or coarse eviction windows), which causes transient OOMs for large tenants and inefficient disk spillback churn. This PR introduces an opportunistic, tenant-aware reclaimer that reduces fragmentation impact and defers destructive merges to a low-pressure background window.

Design summary: single-sentence: background opportunistic relocation + deferred merge + tenant watermarks + spillback backpressure. The reclaimer combines three ideas: (1) cooperative small-object relocation (opportunistic relocation) that sweeps hot/cold boundaries with minimal locking, using a serpentine scanner to avoid long contiguous locks; (2) deferred-merge: when adjacent free segments would normally be merged immediately (causing long critical sections), we mark them for deferred merge and coalesce in a low-priority background pass; (3) tenant-aware watermarks and priority shaping: each tenant has soft/hard watermarks and a small dynamic priority boost if repeated evictions are observed, ensuring fairness under contention.

Key internal changes: moved allocator free-list operations into lock-shard-friendly fast-paths, added a low-priority relocator thread that uses async I/O for spillback, introduced per-tenant reclaim tokens to implement backpressure and to limit concurrent spill ops, and added hooks so the continuous-batching scheduler can query current reclaim pressure to avoid creating large batches that would trigger immediate evictions.

Performance: microbenchmarks + integration: under a 60/40 read/write mixed workload with 120 tenants (skewed Zipf), p50 token latency unchanged, p95 improved 5-8%, p99 improved 9-13% when compared to master at same working set size, overall number of disk spill events reduced ~22%, and the frequency of emergency global evictions (OOME-triggered) dropped from 7 events/hour to 1 event/hour in our cluster repro. Cost: reduced average per-token GPU stalls by ~6% in the end-to-end bench.

Compatibility and rollout: feature gated behind feature_flags.kv_opportunistic_reclaim (default off). Gradual rollout plan: start on a small Dedicated cluster (canary) -> roll to staging where tenant mix is heavy -> enable in production with 1% traffic ramp and metrics guardrail (global eviction rate, per-tenant eviction spikes, p99 tail).

Testing: unit tests for allocator invariants, integration test that simulates tenant skew and ensures no tenant starvation, e2e bench harness added. Bench scripts included for reproducing the reported numbers.

Docs and observability: added a short doc describing knobs and guardrails (docs/perf/kv-reclaim.md) and new Prometheus metrics: kv_reclaimer_cycles_total, kv_deferred_merges_total, kv_reclaim_spillback_ops, kv_tenant_reclaim_tokens.

Rollout notes and known limitations: deferred merges reduce lock contention but may increase short-lived fragmentation under extreme churn — we added an adaptive threshold that forces immediate merge if free-space fragmentation exceeds 12% of slab capacity for a shard. The current implementation prefers low-latency reads; there are trade-offs for very write-heavy tenants that we call out in the docs.

Related work: ENG-4721 (tenant fairness), ENG-4830 (spillback backpressure).
1) add: kv reclaimer core and opportunistic relocator (src/server/kv/reclaimer.cpp, reclaimer.h)
2) perf: integrate reclaim pressure into batch scheduler (src/server/scheduler/batching.cpp)
3) add: per-tenant tokens and watermarks (src/server/kv/tenant_watermark.h)
4) tests: integration + bench (tests/kv/reclaimer_integration_test.cpp, benchmarks/kv/tenant_fairness_bench.py)
5) docs: writeup and rollout instructions (docs/perf/kv-reclaim.md)
6) ci: feature flag and feature gating in deploy (deploy/rollout/feature_flags.yaml)
Ana Rodriguez: Can you explain the choice of deferred merge threshold (12%)? Looks like a magic number — request a short doc or comment in code describing heuristic.
Erin Park: Added comment and a small test that flips the threshold; we tuned it empirically against our staging workload. See reclaimer.h comment and docs/perf/kv-reclaim.md.
Michael Chen: The relocator thread does async spillback — how do we avoid saturating the IO when multiple tenants trigger reclaim concurrently?
Erin Park: We added per-tenant reclaim tokens (default 2) and a global async inflight limit. When tokens are exhausted, reclaim falls back to local eviction (best-effort) and we mark the tenant for a short cooling period. See tenant_watermark.h for parameters.
Priya Nair: CI flake in allocator_shard tests — looks like a race in test harness.
Erin Park: Fixed test harness by adding deterministic seed and stronger ordering in the test; re-ran CI and green.
Added an opportunistic KV reclaimer that reduces fragmentation and emergency evictions by relocating small objects and deferring expensive merges. Feature gated; opt-in rollout recommended. Metrics and docs included.
