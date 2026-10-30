# quantile-driven-partitioner-for-batch-composition-to-minimize-padding

Motivation: continuous-batching in the serving runtime currently groups requests by simple seqlen buckets (power-of-two or fixed ranges). These approaches either over-pad short requests or create too many small batches for long tails, harming throughput and cost. This PR introduces a quantile-driven partitioner that composes batches based on observed sequence-length quantiles per lane/hw-class, combined with an online cost heuristic to minimize padding overhead while respecting latency SLOs.

Design summary: compute lightweight seqlen quantiles over a sliding window per routing lane (e.g. model+device family+priority). Partition incoming requests into quantile bins (Q0..Qk) where k is auto-selected by the estimator to tradeoff batch fill vs queueing delay. At batch flush time the partitioner composes batches by greedily filling each quantile bin up to a kernel-friendly size, but with a kernel-aware cap (knowing which kernel/quantization mode will be used for that seqlen range) so we avoid mixing seqlens that force slow kernels. The system integrates with KV-cache affinity: requests hitting the same cached KV segment are preferred to be co-batched when beneficial; if co-batching would incur excessive padding the affinity is relaxed.

Key algorithmic points: 1) Quantile estimator uses reservoir sampling + fixed-size histogram to remain memory-bounded in high QPS. 2) Auto-k selection uses an AIC-like penalty for partition count to avoid overfitting rare tails. 3) Cost model is additive: token_cost(seq_len) + queue_cost(wait_ms) + kernel_penalty(seqlen_mix). Kernel_penalty is a small precomputed table derived from microbenchmarks (shipping in this PR). 4) Backpressure signals (overall queue depth and model pool utilization) dynamically reduce k to coarsen bins under load.

Why different from existing bucketing: instead of fixed boundaries, quantile bins adapt to workload shifts (e.g., carrier burst of long transcripts) and remove manual tuning per-customer. This complements, not replaces, existing seqlen-bucketing and stride-based batching: it will fall back to coarse buckets if quantiles are unstable.

Benchmarks: microbench on dedicated c5-like and p4d-like nodes with a Llama-2-70b-like trace: median padding tokens reduced by 28% vs current fixed bucketing; P50 latency unchanged, P95 latency improved by 6% on high-mix traces, and per-token GPU time decreased by 12% on average. End-to-end cost modeling on a productionish trace (60M reqs/day synthetic mix) yields estimated 9-14% cost savings depending on kernel choices. More details in notes_on_benchmarks below.

Implementation notes: new files added under src/runtime/batching/* and src/runtime/metrics/*, a small kernel_penalty table generated from scripts in tools/bench/kernel_costs/, and unit + integration tests under tests/batching/. The partitioner is feature-flagged behind FLAG_QUANTILE_PARTITIONER and gated on canary rollout.

Testing & rollout: local unit/integration coverage included, deterministic sim harness reproduced in CI with synthetic traces. Canary rollout plan: 1) enable on 5% of Hosted API traffic for a low-priority model family for 48h, monitor padding_tokens_ratio and latency SLOs; 2) expand to Dedicated pools with canary nodes, 3) global enable for compatible model families. Rollout controlled via Redwood Console flag and canary policy.

Related work: references ENG-8943 (cost-savings initiative), earlier experiments in ENG-7811 (stride-aware batching), and design notes in docs/runbook/batching-heuristics.md.

Checklist: 
- [x] benchmarked on representative traces
- [x] unit & integration tests
- [x] feature flag + canary rollout path
- [x] reviewer approvals

Impact: estimated cost reduction for mixed-seqlen workloads, improved tail latency consistency, and less manual tuning for customers with variable seqlen distributions.
Microbench: Llama-like trace, p95 improvement 6%, padding tokens -28%. Cost model: 9-14% token-cost reduction depending on model/kernel. Bench harness and raw outputs under tools/bench/quantile_results/*.csv
Unit tests for estimator and composer; integration sim reproduces synthetic traffic mixes; smoke canary on dedicated pool; monitor metrics: partitioner.quantile_stability, batching.padding_tokens_ratio, latency.p95, gpu_utilization.
Sofia Patel: Nice work — can you explain how this interacts with continuous batching time-based flush? I worry about increased jitter.
Author (Priya): The partitioner is orthogonal to time-based flush. We maintain per-quantile timers; we queue until either count cap or timer fires. Under high jitter we coarsen bins via backpressure to reduce timer skew.
Ethan Moore: Please add safety to avoid co-batching requests with incompatible KV-cache versions — we shouldn't degrade cache hit isolation.
Author (Priya): Added compatibility check and explicit affinity relaxation policy; see partition_composer.rs change.
Introduce quantile-driven batch partitioner (opt-in). Reduces padding for mixed-seqlen workloads and improves P95 latency in experiments. Feature flagged and rolled out via canary.
