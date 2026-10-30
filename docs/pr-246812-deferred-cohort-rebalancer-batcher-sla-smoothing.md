# Deferred cohort rebalancer: reduce starvation in continuous-batching under skew

Context: under production traffic shapes with long-tailed KV access patterns we observed recurring starvation of low-volume cohorts inside the continuous batcher. Existing admission and slot reservation prevent overtakes but do not address cohorts that slowly accumulate 'stuck' requests because they never hit a favorable batch alignment. Motivation: reduce 99p and tail variance without raising median latency or increasing GPU memory churn. This PR implements a deferred cohort rebalancer that periodically reshuffles low-throughput cohorts into transient priority lanes and applies a capped fast-path kernel hint to shorten their service time, combined with conservative KV-cache pin damping to avoid thrashing.

What this change does (summary):
- Add redwood/pkg/scheduler/cohort_rebalancer.go: core algorithm that tracks cohort age, recent dispatch misses, and fairness score; triggers deferred reassignments when a cohort's starvation_score exceeds a threshold.
- Introduce a lightweight priority lane (continuous-batcher:lane_deferred) that accepts small micro-batches pulled from aged cohorts; these lanes use a short-ttl kernel hint that prefers single-token dispatch paths to lower per-request overhead.
- Modify redwood/pkg/kv/cache_pin.go to add exponential decay to pin weights for cohorts that are temporarily escalated to the deferred lane, preventing long-term cache eviction churn.
- Add instrumentation hooks and metrics: cohort_rebalance.count, cohort_rebalance.latency_gain, cohort_rebalance.pin_decay.events. These feed into existing per-route dashboards.
- Add integration test bench redwood/cmd/bench/continuous_batcher_deferred_test.go and an ops rollout doc docs/ops/cohort_rebalance_rollout.md.

Design notes:
- The rebalancer runs asynchronously and is conservative by default: initial thresholds are tuned to only trigger when a cohort has been delayed for >150ms beyond its expected dispatch window and has at least N=3 outstanding requests. This avoids moving transient spikes.
- To avoid prioritization inversion, a global budget caps concurrent deferred-lane capacity to 6% of GPU slots per worker; the scheduler will drain lanes before creating new deferred micro-batches.
- Kernel hinting is opt-in per model via model manifest; default off for safety. When enabled, hinting picks a lightweight kernel path (reduced kv prefetch, contiguous attention path) that reduces single-request overhead but may slightly lower throughput for very large batches.

Testing and benchmarking:
- Unit tests: 28 new unit tests covering starvation scoring, decay math, and lane lifecycle.
- Integration: rerunnable bench harness replicates production skew (90/10 hot/cold key split) across 4 replicates. Results: 99p latency reduced by 25-40% on KV-heavy scenarios; median latency change within +/-3%. CPU/GPU and KV cache eviction traces show reduced pin churn when decay is enabled.
- Canary rollout plan: staged rollout on Dedicated pools with feature flag redwood.scheduler.deferred_rebalancer, 1% -> 10% -> 50% -> 100% over 72h with automated rollback if 99p latency regresses >10% or error rate increases by >0.5%. See docs/ops/cohort_rebalance_rollout.md.

Related: ENG-2468 (continuous batching SLA hardening), ENG-2471 (kv-cache pin stability).

Checklist:
- [x] Design doc reviewed by runtime team
- [x] Benchmarks attached and reproducible in bench harness
- [x] Unit and integration tests added
- [x] Rollout plan included

Notes: This is intentionally conservative; follow-up PR planned to add adaptive threshold tuning based on live latency signals and to expose knob controls in the Console for enterprise tenants.
Introduce a deferred cohort rebalancer to improve fairness and reduce request starvation in continuous batching. Microbenchmarks show 25-40% reduction in 99th percentile latency under heavy skewed KV-heavy workloads.
Liam Chen: Could you add a guard for models with extremely large attention memory so the kernel hint doesn't regress throughput? — Addressed: added model-manifest opt-in and a runtime size check.
Priya Nair: Need more visibility into cache pin decay parameters for SRE teams. — Addressed: exposed decay alpha and ttl via metric tags and docs/ops page.
Diego Alvarez: Is the 6% slot cap empirical? Would like a link to microbench results. — Reply: yes, included bench results in the PR; cap is conservative and will be adjustable via runtime flag.
