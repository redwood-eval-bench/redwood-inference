# Intelligent admission window oscillation damper and KV-prefetch hinting

Motivation: Continuous-batcher admission windows exhibit oscillatory behavior under mixed workloads (large KV-heavy requests + latency-sensitive small requests). This can cause bursts of large batches that briefly monopolize slots and trigger downstream KV cache churn which starves small requests and increases P95/P99 latency. We also observed runaway requeueing when admission decisions flip quickly due to tight timeouts.

Goal: Reduce admission oscillation, protect short-latency traffic from transient starvation, and add a lightweight hint to the KV cache warmup to avoid unnecessary prefetch thrash. The change must be conservative (no throughput regression for batch-friendly workloads) and observable.

What this PR does: 1) Introduces an "admission damper" state machine in the continuous batching scheduler that applies exponential dampening to aggressive window enlargements and constrains rapid downsizing within a short horizon. 2) Adds jittered soft-timeouts to admission probes (to avoid synchronized flips across worker threads). 3) Implements a soft reservation slot for latency-sensitive classes (configurable via route metadata) that decays gracefully to avoid long-term throughput loss. 4) Adds a KV-prefetch hint emitted at admission time for batches that appear to include cold KV patterns; the runtime uses this hint to issue coalesced prefetches without forcing synchronous fetches. 5) Instrumentation: new metrics (scheduler.admission_damp_events, scheduler.soft_reserve_hits, scheduler.prefetch_hints_issued, scheduler.oscillation_index) and a small diagnostics trace.

Implementation notes: - admission damper is implemented in src/scheduler/admission_damper.rs and wired into the continuous batcher loop. The damper keeps a short history (last 12 admissions) and computes a lightweight oscillation score; when the score crosses a small threshold the damper reduces admitted window delta by a damp_factor (configurable) for a recovery period. - soft-timeouts live in runtime/batcher/timeout_backoff.cc and use randomized jitter based on request IDs. - soft reservation is enforced via a probabilistic ticket allocation in src/scheduler/soft_reserve.rs; we avoid hard partitions by letting reserves decay based on observed occupancy. - KV-prefetch hint is an opaque flag set in the batch descriptor; the KV prefetcher module coalesces hints into 5–10ms micro-prefetch windows, no synchronous waits added. - Tests: unit tests for damper logic, integration test simulating mixed traffic with synthetic KV cold lists, and a bench harness to validate no throughput regression.

Checklist: - [x] Add admission damper logic + unit tests - [x] Add jittered soft-timeouts - [x] Soft reservation for latency-sensitive routes - [x] KV-prefetch hint + coalescer - [x] Add metrics and basic trace points - [x] Bench harness and no-regression runs

Backward compat: Defaults keep previous behavior (damper disabled unless route-level config or global flag enabled). KV-prefetch hint is opt-in behind a config flag. No breaking config changes.

Performance summary (from bench harness): 1) Mixed workload (80% small, 20% large KV requests): P95 reduced by ~18%, P99 reduced by ~12% under medium load. 2) Throughput on batch-friendly workloads: <1% difference. 3) Median latency unchanged for small requests. Detailed results attached in CI artifacts (bench/2025-11-02).

Notes and next steps: - Consider adaptive damp_factor tuning using reinforcement signals (follow-up). - Expose soft-reserve percentage via Console and add an SLO guard for reserve-backed routes.

Related: ENG-4203 (scheduling stability), SUP-1223 (customer-reported KV starvation incident).
Aisha: Can we make the damp_factor per-route? Some enterprise customers will want fine control.
Author (Priya): Damp factor is currently global with an override via route metadata. Added docs note and config example.
Marco: The soft-reserve could cause long-term unfairness if route annotation is misused. Suggest adding a monitoring alert for reserve saturation.
Author (Priya): Added scheduler.soft_reserve_hits metric and an alert proposal in docs/observability/scheduler-reserve.md
Leah: Please ensure jitter range is small enough to not blow up P50. Also add deterministic seed in unit tests.
Author (Priya): Unit tests use deterministic PRNG seed; jitter range validated in bench runs.
Aisha: CI flake on integration test in initial run (race on metrics registration). Fixed by lazy-init and rebase.
Marco: Approved after bench artifacts validated.
(approved-reason: observed P95/P99 improvements for mixed workloads and negligible throughput impact)
scheduler: Add admission-damper to reduce continuous-batcher oscillation and protect latency-sensitive traffic. Add optional KV-prefetch hinting to reduce cold-KV induced latency spikes. Metrics added: scheduler.admission_damp_events, scheduler.soft_reserve_hits, scheduler.prefetch_hints_issued.
This PR is intentionally conservative and gated behind route/global flags. Follow-ups planned to surface reserve controls in Console and to experiment with auto-tuning damp factors.
