# probabilistic-flush-and-microbatch-prioritization-to-cut-streaming-first-byte-latency

Summary: This PR introduces a probabilistic flush scheduler and microbatch prioritization in the serving runtime to reduce streaming first-byte latency for small, latency-sensitive requests while retaining batching efficiency for throughput workloads.

Motivation: recent production traces (ENG-4821) showed a consistent 40-120ms tail on first-byte for short responses when continuous batching held tokens waiting for batch fullness. We need a low-risk change that improves p50/p95 for small messages without regressing cost/throughput for bulk workloads.

Design: 1) ProbabilisticFlushScheduler: a scheduler that, for a microbatch below a configurable size threshold, decides to flush immediately with probability p(t) where p increases with queued duration. The stochastic decision reduces tail waiting by opportunistically releasing ready tokens while keeping some batching benefits. The function p(t) is configurable and defaults to a conservative curve tuned from production traces. 2) MicrobatchPriorityQueue: small requests are assigned elevated scheduling weight so that KV prefetch and kernel allocation favor them for the next scheduling slot. This prevents tiny microbatches from being starved by a stream of large batched requests. 3) KV prefetch window tuning: prefetch window is shortened for prioritized microbatches, and an inline optimistic prefetch path was added to reduce KV cache miss latency on first token. 4) Kernel scheduling hints: added light-weight kernel pinning hints so prioritized microbatches attempt to reuse a warm kernel when available.

Implementation details: - runtime/streaming/probabilistic_flush_scheduler.cc: new scheduler implementation and configuration knobs. - runtime/scheduler/microbatch_priority_queue.h/.cc: new priority queue, lightweight comparator based on size, age, and priority class. - runtime/kv/prefetcher.rs: added optimistic inline prefetch path with a fast-path check to avoid allocations on hits. - runtime/kernel/hints.h: small hint enum and runtime sampling counters. - dispatch tests and integration harness updates under e2e/streaming to validate p50/p95.

High-level commits: - Add probabilistic flush scheduler and config defaults - Add microbatch priority queue and wire into stream dispatcher - Add KV optimistic prefetch path and adjust prefetch window tuning - Add kernel hinting and counters - Tests: unit, integration, and scaled canary harness

Changes at a glance (high-level file list): - runtime/streaming/probabilistic_flush_scheduler.cc - runtime/scheduler/microbatch_priority_queue.h - runtime/scheduler/microbatch_priority_queue.cc - runtime/kv/prefetcher.rs - runtime/kernel/hints.h - e2e/streaming/canary_runner.py - tests/streaming/test_flush_behaviour.cc - docs/ops/flush_tuning.md - examples/streaming/flush-config.yaml

Behavioural knobs (configurable): - streaming.flush.prob_curve: base/progression curve (default conservative) - streaming.flush.size_threshold: microbatch size threshold for eligibility - streaming.priority.size_cutoff: what counts as small request - streaming.kv.prefetch_window_ms: target prefetch window for prioritized microbatches

Performance results (canary): - Canary run on a mixed workload (50% small prompt < 32 tokens, 50% large throughput batches) shows: p50 first-byte improved from 38ms to 19ms, p95 improved from 126ms to 68ms for small prompts. Overall token throughput decreased by ~1.8% in the synthetic mix, within tolerance for our Dedicated customers. Cost impact is expected to be neutral at scale because probabilistic flush preserves batching for most requests.

Safety and rollout plan: - Default conservative probability curve and size thresholds to avoid aggressive flushes. - Feature gate: streaming.enable_probabilistic_flush (off by default for private deployments). - Gradual rollout: canary -> regional rollout -> global.

Observability: added new metrics: streaming.flushes.probabilistic, streaming.microbatch.priority_skipped, kv.prefetch.optimistic_hit_rate, kernel.hint_reuse_rate. Added corresponding dashboards and alerts (ops doc referenced below).

Tests: - Unit tests for scheduler determinism with mocked time. - Integration test in e2e harness emulating mixed latency/throughput workloads. - Stress test for KV optimistic prefetch path to validate stable memory behavior.

Review conversation (selected): Reviewer Aisha Patel: "High level: solid approach, but can you justify the probabilistic curve shape? Concerned about oscillation under bursty arrival patterns." Author reply (Priya Rao): "Added a doc with empirical curve fit from production traces and a safety backoff when host CPU or kernel contention is detected. Will include adaptive backoff in follow-up if needed." Reviewer Marco Li: "Please add observability for flush decisions and make sure the metric cardinality is bounded." Author reply: "Added streaming.flush.decisions.count with labels limited to {probabilistic, forced, scheduled}. Kept label cardinality to 3." Reviewer Evan Hart: "Stress tests look good, but add a test that simulates many small requests interleaved with huge batches to ensure no starvation." Author reply: "Added tests/streaming/test_priority_starvation.cc and validated locally."

Requested changes and resolution: - Reduce default base probability by 20% (addressed). - Expose configuration via runtime flags and console UI (console change tracked in ENG-4893; small docs stub included). - Add canary gating flag for Dedicated (added).

CI and checks: - CI initially failed one flaky test in tests/streaming/test_flush_behaviour.cc (timing sensitive). - I fixed the test by using an event barrier and re-ran CI. Final CI: build: pass, unit: pass, integration: pass, e2e-canary: pass.

Merge notes: - Squash-merged after approvals from Aisha and Marco and green CI. - No database or API contract changes.

Ops notes and release_notes: "Improves streaming first-byte latency for small prompts via probabilistic flush and microbatch prioritization. Default settings conservative; operators can tune via streaming.* flags. See docs/ops/flush_tuning.md for details."

Backcompat: This is a non-breaking runtime improvement behind gates; behavior is opt-in for Private by default and conservative for Hosted.

Risks and mitigations: - Risk: slight throughput degradation in mixed workloads; mitigated by conservative defaults and canary rollout. - Risk: metric noise; mitigated by bounded metric labels.

Follow-ups: - Add adaptive probability adjustment based on observed queue length and kernel contention (planned ENG-4992). - Surface a per-customer tuning preset for Dedicated customers.

Example config (examples/streaming/flush-config.yaml): "streaming:\n  enable_probabilistic_flush: true\n  flush:\n    size_threshold: 8\n    prob_curve: \"0.01,0.05,0.15,0.4\"\n    kv_prefetch_window_ms: 10"

Acknowledgements: thanks to Marco for kernel hint guidance and Evan for stress harness improvements.
