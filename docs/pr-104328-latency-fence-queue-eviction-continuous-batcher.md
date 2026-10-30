# introduce latency-fence admission + queue-eviction policy for continuous batcher

Added a latency-fence admission controller and a queue eviction policy for the continuous batcher to reduce tail latency regressions under high KV-pin contention. No API changes. Rollout gated behind feature flag.
Motivation: In production we observed recurring tail latency spikes when there is a mix of KV-heavy long-context requests and short low-cost requests. Existing continuous-batcher admission prioritizes throughput via coalescing which can let large KV-pin requests monopolize slots and cause starvation of short requests and SLO violations for p99. This PR introduces a lightweight runtime admission+eviction mechanism (latency-fence) that enforces a moving latency budget per scheduling horizon and allows preemptive eviction of low-priority queued items when predicted p99 would be violated.

What this change does (high level):
- Adds a LatencyFence admission controller to the continuous batcher that maintains an exponential moving window of recently observed tail latencies (configurable percentile and decay). Incoming requests are classified by cost category (short/medium/kv-heavy) using existing runtime hints plus a small learned weight table. If admitting a request would push the predicted percentile above the configured fence, the controller will either: (a) reject/admit-to-fallback (for hosted paths), (b) mark for early eviction from the queue in favor of short requests, or (c) stall admission and trigger a fast-drain policy depending on the policy mode.
- Implements QueueEvictor which performs bounded, fairness-aware eviction during scheduling ticks. Eviction picks candidates by age/x-penalty and uses a per-queue debt token to prevent repeated eviction of the same tenant (debt tokens decay over time).
- Adds probabilistic KV-pin preemption: for requests that hold large KV pins but are stuck behind in waiting queues, the scheduler now opportunistically moves their KV pin into a compacted spill buffer (best-effort, does not alter model correctness) to allow serving of short requests first. This is guarded by config and telemetry to make it conservative.
- Adds a percentile-based timeout guard that will fast-fail queued requests older than N * EMWA(p99) to avoid queue buildup cascading into long tails. This is a soft fail when fallback is available; otherwise it extends the deadline with backpressure token.
- Exposes metrics (batcher.latency_fence.{admits,evictions,preemptions,fastfails}, batcher.eviction_age_dist) and a new per-route config flags in serving.yaml for experiment rollouts.
- Small kernel hint: when QueueEvictor triggers, it hints a short-lived kernel "kv_compact_hint" so the runtime selects a lower-latency kernel path for spill operations.

Files changed (high level):
- pkg/serving/batcher/latency_fence.go (new): core controller and config.
- pkg/serving/batcher/evictor.go (new): eviction selection and debt-token logic.
- pkg/serving/batcher/scheduler.go (modified): integrate admission checks and eviction hooks; add metrics emission.
- pkg/serving/kv/spill_buffer.go (modified): compact spill buffer implementation + kernel hint interface.
- pkg/serving/config/serving.yaml (modified): new per-route flags and defaults.
- internal/e2e/batcher_latency_test.go (new): deterministic scenarios for starvation and preemption.
- benchmarks/batcher/bench_latency_fence.go (new): microbench and p99-oriented bench harness.
- docs/observability/batcher_metrics.md (modified): documentation for new metrics.

Commits (summary):
- 8c7d3a2 add LatencyFence controller and basic EMWA p99 store
- 1b9e5f4 implement QueueEvictor and debt tokens with tests
- f204b11 hook admission checks into scheduler tick and add metrics
- a64f3d0 add KV spill buffer compact path + kernel hint API
- d1b4e9c add e2e tests and bench harness; docs for metrics
- c3b2a76 tweak thresholds after perf run, add feature flag gating

Design notes and tradeoffs:
- This PR is intentionally conservative: evictions are bounded and debt tokens are used to avoid oscillation and tenant starvation. We prefer fast-fail + fallback over hard eviction when fallback model variants exist.
- The EMWA p99 prediction is lightweight and tuned for responsiveness rather than absolute accuracy; it worked well in lab workloads and reduced p99 in our canary by ~18-22% in mixed KV/short workload at 95th percentile load.
- Probabilistic KV-pin preemption is best-effort: it can reduce per-token CPU by moving pinned KV segments into compact buffers, but we do not change model outputs. This avoids complexity of reloading weights.

Benchmarks:
- microbench (bench_latency_fence) shows p95/p99 improvements in mixed workload (25% p95, 18% p99) at target throughput with config fence=95, decay=0.2.
- End-to-end canary (10% of traffic for two hours) against main reduced observed p99 tail spikes from 340ms->270ms and reduced queued request count by ~30%.

Tests:
- unit: evictor selection, debt token decay, EMWA updates.
- integration: scheduler admission flows, kernel-hint path executed when spill takes place.
- e2e deterministic test that simulates KV-heavy tenant plus bursty short requests to validate fairness and that debt tokens prevent livelock.

Observability and rollout plan:
- Feature gated behind serving.yaml: routes can opt-in to "latency_fence: enabled|experiment|disabled". Default is disabled.
- Metrics added to Console dashboards and Alert rule for spike in batcher.latency_fence.evictions or fastfails.
- Rollout plan: enable for internal canary tenants (dedicated capacity) -> 5% hosted traffic -> 25% -> wider. Watch p99, evictions, preemptions, and errors.

Backward compatibility: No public API changes. Configs default to off. If teams enable the feature they should verify fallbacks are available for critical routes.

Reviewer discussion (highlights):
- Diego Alvarez: "Nice focused work. Need to ensure debt token rate is safe under tenants doing repeated short bursts. Please add amortized tests."
  - Author: "Added debt-token amortized decay test and extended e2e to simulate oscillating tenants."
- Carmen Liu: "Concerned about false positives from EMWA p99 on cold starts. Can we add a startup grace period?"
  - Author: "Added startup grace flag (default 60s) and unit tests covering it."
- Maya Kapoor: "Can we gate kernel hint behind runtime capability detection? Not all runtimes support the kv_compact_hint."
  - Author: "Added capability probe at scheduler init and no-op fallback when unsupported."

CI summary:
- unit: pass (ran on linux/amd64 and linux/arm64)
- integration: pass (flaky test patched during review and re-run)
- e2e: pass (deterministic harness)
- bench: bench job completed; artifacts uploaded with latency histograms.

Merge notes:
- Merge method: squash
- Changelog: included under release_notes above.
- Post-merge: enable feature-flag in staging for a 24h soak and add dashboard panels for the new metrics. We will coordinate with SRE to set preliminary alerts.

Potential follow-ups (next PRs):
- refine admission prediction with small online learner per-route
- make eviction policy aware of route SLAs to support strict tenant-level SLOs
- explore kernel-level optimizations to speed spill compaction paths further

Contact: @asha (author) for rollout coordination and benchmarks.
