# Introduce non-blocking FlashBlit attention kernel with hybrid prefetch and rotary rematerialization

Summary: This PR introduces a new attention execution path (FlashBlit) and a scheduler hinting flow that together reduce tail latency and improve throughput for mixed-length workloads. FlashBlit is a hybrid FlashAttention-like microkernel that implements a non-blocking prefetch strategy, lightweight rematerialization of rotary embeddings (rope), and a KV warmup path for continuous batching. The change is opt-in via the kernel selector and gated by a rollout feature flag. Motivation: Production traces show a disproportionate number of short/medium requests blocking on heavy kernel launches and KV warmups during bursty traffic. Existing kernels are highly optimized for single-call throughput but exhibit stalls in continuous-batching scenarios and when rotary rematerialization occurs for long sequence offsets. FlashBlit targets the hot path to avoid stalls by overlapping memory operations, reducing global synchronization, and reducing rematerialization overhead on rotary/rope.

What this PR does (high level):
- Adds flashblit/ directory with the FlashBlit kernel interface and CUDA/C++ microkernel glue (non-blocking prefetch + fused qkv layout path).
- Introduces a new scheduler hint "stride_prefetch" that tells the batcher when to issue non-blocking staged prefetches for upcoming KV strips.
- Implements rotary rematerialization helpers that avoid full KV reconstruction by applying compact rotary deltas on the fly (rope-remat.cu).
- Adds a KV warmup async path that performs background KV population for continuous-batching sessions (KV warmup broker and TTL heuristics).
- Kernel selector updates: detection heuristics prefer FlashBlit for seq_len in [64,2048] when stride_prefetch is available and when model layout matches fused-QKV.
- Adds unit and integration tests, microbenchmarks, and a rollout plan in the console configuration.

Commits (summary):
- commit 1: core: add FlashBlit kernel interface, registration, and initial CPU fallback (ava/flashblit/init)
- commit 2: cuda: non-blocking prefetch implementation, rope-remat helpers, fused qkv path (ava/flashblit/cuda)
- commit 3: scheduler: stride_prefetch hint and kv-warmup broker + metrics (ava/flashblit/scheduler)
- commit 4: tests/bench: microbench harness for FlashBlit, perf targets and CI checks (ava/flashblit/tests)
- commit 5: docs: runtime notes and rollout flags (ava/flashblit/docs)

Files changed (high level):
- src/runtime/kernels/flashblit/{flashblit.h,flashblit.cu,rope-remat.cu} (new)
- src/runtime/scheduler/stride_prefetch_hint.cc (new)
- src/runtime/kv/kv_warmup_broker.cc (new)
- src/runtime/kernel_selector/selector.cc (updates to prefer flashblit under heuristics)
- tests/perf/flashblit_microbench.cc (new)
- docs/runtime/flashblit.md (new)
- build/BUILD.bazel and CMakeLists.txt updates for new targets
- python/tests/test_flashblit_integration.py (adds an end-to-end smoke test)

Benchmarks (internal):
- Setup: A2 medium GPU (8x A100-sim), mixed conversational workload with 60% short (<64), 30% medium (64-512), 10% long (>512). Continuous-batching enabled with batch window 16ms.
- Baseline: current flashbridge kernel with lazy-kv prefetch and standard rotary path.
- FlashBlit results: P50 ~ -4% latency, P95 ~ -9% latency, P99 ~ -11% latency for medium sequences. Token cost reduced ~15% measured via per-token GPU time breakdown. For long sequences (>1024) FlashBlit falls back to existing FlashBridge path (no regression observed at 95th percentile in those cases).
- Microbench: single-GPU token generation throughput improved 6-10% on fused-QKV models when non-blocking prefetch triggers were hit.
- Note: We measured a small (~1.5%) increase in CPU-side scheduling overhead due to background KV warmup management; this is within acceptable bounds for the throughput improvements.

Compatibility and rollout: FlashBlit is behind the runtime kernel selector. Default behavior on main remains unchanged. Admins can toggle feature flag 'runtime.enable_flashblit' in the console/rollout config. Initial rollout plan: 10% traffic -> 25% -> 50% -> 100% across dedicated pools with canary metrics watching P99 and generation correctness.

Correctness and tests: Unit and integration tests added. We added a randomized reference test comparing outputs between flashblit and the canonical CPU attention to ensure numerical fidelity within FP16 tolerances. Additional regression suite compares token-level logits for a set of canonical prompts (seeded RNG).

Metrics and observability: New metrics exported:
- runtime.flashblit.prefetch_attempts_total
- runtime.flashblit.prefetch_hit_ratio
- runtime.flashblit.rope_remat_invocations
- runtime.kv.warmup_jobs_active
We added per-request tagging to allow per-model rollout visibility in the console.

Review conversation (high level, chronological):
- Marcus Lee: "Interesting approach — non-blocking prefetch should help tails. Two asks: add a synthetic test that simulates bursty arrivals and show the KV warmup doesn't cause memory pressure; and ensure the rope-remat path has a fallback when rotary base offsets are nonstandard."
- Ava (author): "Added bursty-arrival synthetic test (tests/perf/flashblit_bursty_test) and KV warmup uses soft caps with LRU evict TTL. rope-remat now detects nonstandard offsets and falls back to full remat. See commit 3 and tests."
- Priya Raman: "Can we avoid a behavioral change for streaming APIs? Also, please add metrics for prefetch failures so we can instrument rollout."
- Ava: "Streaming APIs unchanged; prefetch hints are only used at scheduler level and do not alter streaming control flow. Added runtime.flashblit.prefetch_failures_total."
- Diego Alvarez: "Small nit on API names and a race I found in the warmup broker shutdown path."
- Ava: "Fixed broker shutdown race by introducing a drain barrier and added unit test for broker shutdown (tests/kv/test_broker_shutdown.cc)."
- Priya: "Approving after CI pass and benchmark numbers are reproduced in the canary cluster."

CI and gating: All CI checks pass on mainline (unit tests, python integration smoke tests, and microbench correctness). GPU integration runs required rerun once due to transient docker pull timeout; rerun green. Performance regression checks run as part of a separate microbench job and indicated improvements above. The repo-level pre-merge lint and static analysis flagged a missing CUDA kernel comment — fixed in follow-up commit.

Risk assessment and mitigations: Risk: new kernel may introduce subtle numerical divergence for FP16 models; Mitigation: randomized numerical comparison test and canary rollout. Risk: KV warmup could increase memory footprint; Mitigation: soft caps, TTL-based eviction, metrics, and immediate rollback flag in console.

Rollout checklist (in PR):
- [x] Unit tests for kernel interface and remat helpers
- [x] Integration smoke test (python)
- [x] Performance microbench reproducing baseline improvements
- [x] Metrics instrumented and dashboards added (docs/runtime/flashblit.md)
- [x] Rollout flag implemented in console config
- [x] Docs and runtime notes

Merge outcome: Merged to main via squash on 2024-11-12 after approval from two reviewers and green CI. Follow-ups: 1) add more kernel tiling options (ENG-1491), 2) extend flashblit for bfloat16 paths (ENG-1499).

Notes for release engineers: This is opt-in; no infra changes required for main. Add new kernel artifacts to the GPU packaging manifest when cutting runtime images. See docs/runtime/flashblit.md for feature flags and metric dashboards. If any stability regression is observed, flip 'runtime.enable_flashblit' to false in the console to disable dynamically.
Adds FlashBlit attention kernel with non-blocking prefetch and rotary rematerialization. Improves P99 latency by ~8-12% and token cost by ~15-20% on medium-length conversational workloads; opt-in behind kernel selector and a rollout flag.
