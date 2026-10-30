# Introduce stream-shepherd credit shaping and kernel throttling to smooth token emission

Motivation: customer telemetry and internal canaries showed intermittent first-byte jitter in streaming responses under moderate bursty load. Root cause was interaction between instantaneous kernel dispatch (bursty GPU occupancy), KV cache contention on hot prefixes, and unbounded token emission when credits are returned immediately. This PR implements a lightweight controller ("stream shepherd") that shapes outbound token credits per-connection using a latency-budget-aware window and applies soft kernel throttling to avoid instant GPU occupancy spikes.

What this changes: 1) Credit shaping: per-stream credit windows are capped by a latency budget estimator and an adaptive refill timer. Credits returned from the generator are bucketed into a sliding token window (configurable, default 25ms) instead of being emitted immediately. 2) Kernel throttling: a soft rate limiter that hints to the scheduler to spread kernel launches across a small time jitter when the estimated tail pressure is above threshold. This avoids synchronous kernel spikes when many small streams finish KV lookups simultaneously. 3) KV prelock warm: add an opportunistic short-term KV lock for confirmed hot prefix hits to reduce high-contention stalls during window refill. This is conservative — it only applies when prefix hit rate > 70% and path latency < configured threshold. 4) Sizing controller: continuous-batching controller tweaks microbatch sizes up/down within a narrow band to keep latency within SLO while maximizing throughput.

Checklist: [x] Design doc + microbench plan (linked in Linear ENG-4921), [x] Unit tests for credit window logic, [x] End-to-end canary in benchmark-lab, [x] Safety flag gated behind feature flag `stream_shepherd_v1`, [x] Metrics and dashboard additions, [x] Rollout plan with staged percentage.

Testing and results: - Unit tests cover window arithmetic, refill cooldown, and kernel throttle heuristics. - Integration: ran a 48-hour canary against a mixed GPT-like 8k-seq workload (2.5k RPS emulator). Observed median TTFB improvement of 18% and p95 improvement of 12% for streaming routes under moderate burstiness. CPU/GPU utilization smoothed (less 99th-percentile GPU launch spikes). - Benchmarks: no regression on throughput for long-running steady-state throughput tests; microbatch controller increased average batch size by ~6% without SLO violations.

Rollout: feature gated; enable 5% traffic -> 25% -> 100% with checkpoints at each stage for regressions. Canary dashboards added: stream_shepherd.tokens_emitted, stream_shepherd.window_fill_latency_ms, scheduler.kernel_throttle_ratio.

Operational notes: feature can be disabled at runtime. Default behavior is unchanged when the flag is off. Initial default parameters are conservative (window=25ms, throttle_threshold=0.08 tail pressure). We will tighten after broader telemetry.

Related: ties to ENG-4980 (scheduler heuristics) and a follow-up ticket to explore making the latency-budget estimator cross-tenant (ENG-5050).
Omar: overall looks good; can we make the default window configurable via runtime config? (answered: added runtime flag, default 25ms)
Linnea: ensure this doesn't starve low-rate streams — suggested a minimum token allowance per stream (implemented)
Amit: questioned the KV prelock safety; added extra guard and TTL for the short-term lock to be conservative
Add an opt-in stream-shepherd controller that smooths token emission and reduces streaming first-byte jitter. Enabled behind feature flag. Improvements expected in median and tail TTFB for bursty workloads with minimal throughput impact.
