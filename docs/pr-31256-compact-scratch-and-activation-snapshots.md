# Compact scratch allocator and activation snapshotting to reduce GPU workspace

Motivation: production multitenant inference workloads showed sporadic OOMs on medium-sized GPU instances when prefix caching and long-context runs were combined with background continuous-batching. This PR introduces two orthogonal memory-reduction techniques: 1) activation snapshotting — a lightweight, eviction-friendly checkpoint that stores compact metadata for activations to allow later partial recomputation instead of holding full activation slabs, and 2) a scratch compactor for per-request temporary buffers that reduces fragmentation and enables coalescing of short-lived scratch allocations across micro-batches. The changes also include scheduler tuning to align KV-cache shard residency with compacted scratch windows so that hot shards are kept contiguous and do not force extra workspace growth. Impact: in our microbench harness (multi-stream, 8 concurrent streams, auto-batching) peak workspace on an A100-40GB fleet dropped ~18% and average per-request GPU memory dropped ~12%, with tail latency unchanged (+0.4ms median). The PR is conservative by default: both features are opt-in via runtime flags (enable_activation_snapshot, scratch_compaction_level) and fall back to current checkpointing when models request strict deterministic execution. Includes unit and perf tests, canary rollout plan in release notes.
Init: add snapshot data structures and compacted slab allocator interface
Implement: scratch_compactor with greedy-pass and occupancy map
Scheduler: align KV-shard placement to compacted scratch windows
Runtime: expose flags and plumbing for activation snapshots and compactor
Tests: add microbench harness and regression tests; fix memory leak in allocator destructor
Docs: perf notes and rollout steps
Diego Ramos: Nice improvement; can we get numbers for large-sequence (8k) generation? Also add a safety check to avoid snapshotting when deterministic seeds are required.
Marina Kline: Added 8k-generation numbers to docs/perf report and added a runtime guard (preserve_det_state) that skips snapshotting when true.
Anika Shah: There was a flaky test on CI (allocator_dtor on ppc64). I suspect a missing fence in free path.
Marina Kline: Reworked free path with explicit device synchronization fence on platforms that require it, reran CI; test stabilized.
Priya Menon: Approving pending a short changelog blurb and a canary rollout plan for Dedicated customers.
Marina Kline: Added release notes & canary rollout to docs/perf/reports and updated CHANGELOG entry.
Add opt-in activation snapshotting and scratch compaction to reduce GPU workspace and fragmentation. Default behavior unchanged; opt-in flags: enable_activation_snapshot, scratch_compaction_level (off|conservative|aggressive). Reduces peak workspace (~18%) and avg per-request GPU memory (~12%) for multi-stream workloads in internal microbench. See docs/perf/reports/2026-02-snapshot-compactor.md for details and canary rollout plan.
