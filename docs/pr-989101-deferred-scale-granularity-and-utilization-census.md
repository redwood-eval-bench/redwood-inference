# Deferred scale granularity and low-overhead utilization census

Motivation: frequent fine-grained scale events on Dedicated clusters cause allocation churn and transient latency spikes for steady sustained workloads. This PR introduces two coordinated changes: (1) a deferred scale granularity layer (Scale Decoupler) that groups candidate scale actions into a configurable time bucket and emits a single reconciled action to the provisioner, and (2) a low-overhead utilization census service that samples per-pool signals with adaptive sampling to provide a robust decayed-utilization picture for the autoscaler. Both changes aim to reduce flapping, improve tail latency during scale transitions, and preserve burst capacity guarantees.

What changed (high level):
- New Scale Decoupler component in controller plane that buffers per-tenant and per-pool scale intents for a configurable window (default 30s) and merges them using policy heuristics (favor sustained growth, respect emergency fast-paths).
- Utilization Census daemon that replaces expensive full-pool scans with lightweight randomized probes + exponential decay windows; exports cohorted histograms and percentile summaries to the metrics pipeline.
- Autoscaler updated to query Census summaries and to respect decoupler-recommended consolidated actions; introduces hysteresis knobs and a min-delta threshold to avoid sub-token scale churn.
- Telemetry: added ring buffer counters, 'census.sample_rate' and 'decoupler.emitted_actions' metrics, and tracing spans for the merge decision path.
- New unit and e2e tests for decoupler merge semantics and census sampling accuracy; stress test added to perf-canary harness.

Checklist:
- [x] Design doc linked in description (Linear ENG-2247)
- [x] Unit tests for merge policy and utility functions
- [x] Integration tests for end-to-end scaling on a small dedicated cluster
- [x] Metrics and traces wired to observability console
- [x] Backwards-compatible (decoupler is opt-in, census can fall back to full-scan)

Impact: rolling this out cluster-by-cluster should reduce the number of scale actions per hour for steady workloads by ~60% in our internal perf runs, reduce allocation churn and lower 99th percentile latency during scaling by ~8-15%. The change preserves existing SLA fast-paths (emergency preemption when backlog exceeds threshold).
Mateo Ruiz: Nice isolation between decoupler and provisioner. Request: expose a metric for 'merged_action_size' to monitor consolidation ratios.
Asha Menon (author): Added 'decoupler.merged_action_size' histogram in the latest commit and documented threshold implications.
Lena Cho: Census looks promising, but sampling rate defaults may be aggressive for very small pools. Please add a config knob and a fallback to full-scan for pools < 5 nodes.
Asha Menon: Added 'census.min_pool_size' config and a small-pool fallback path, plus unit tests that cover the behavior.
Dmitri Petrov: Found a race in merge decision under concurrent intent updates (flaky in integration). Suggest using a compare-and-swap guard or serializing per-pool intent commits.
Asha Menon: Reworked intent buffering to use per-pool mutexes and added deterministic test harness; integration flakiness resolved.
Mateo Ruiz: CI green, approving for canary. Please add a rollout plan in the ops doc before GA.
Lena Cho: Approved after small nits fixed in docs.
Introduce Scale Decoupler (opt-in): groups scale intents to reduce churn and allocation noise.
Introduce Utilization Census: low-overhead adaptive sampling for per-pool utilization reporting.
Autoscaler now consults census summaries and respects decoupler-consolidated actions; conservative defaults ensure backwards compatibility.
New metrics: decoupler.emitted_actions, decoupler.merged_action_size, census.sample_rate.
