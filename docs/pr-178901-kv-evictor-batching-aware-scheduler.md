# KV: introduce batching-aware eviction scheduler with latency budgets

Motivation: In production we observed tail-latency regressions under mixed workloads where continuous batching and reactive KV cache eviction interacted poorly. Large synchronous spill/compaction work would be scheduled at the same time as batch flushes, causing per-request blocking inside the serving path and +250-400ms tail spikes for a subset of requests. The root causes were: (1) eviction/spill tasks were triggered from request hot paths and could preempt batch assembly; (2) eviction prioritization was purely capacity-driven and did not respect latency SLO budgets; (3) the runtime kernel selection had no visibility into concurrent cache IO pressure, leading to suboptimal long-kernel choices during heavy eviction.

Proposal: Add a batching-aware eviction scheduler that (a) separates eviction initiation from request threads, pushing eviction to background workers with explicit latency budgets, (b) tracks per-tenant batch windows and delays non-critical eviction work until after in-flight batch assembly completes when safe, (c) exposes an eviction pressure signal to the scheduler that the kernel selector can consume to prefer low-latency kernels during high IO pressure, and (d) implements a predictive hot-range prefetch / short-lived pin to reduce thrash when evict/restore cycles detect reaccess patterns. This change aims to reduce tail-latency without increasing steady-state memory pressure by making eviction more cooperative with continuous batching.

What I changed: Implemented a new component kv/evict/scheduler.go (BatchAwareEvictor) and supporting queue primitives. Added a lightweight eviction-pressure metric and API that kernel_selector consults to bias kernel choice. Introduced batch-window markers that the serving runtime writes when a continuous-batching window opens/closes; evictor observes markers and delays non-urgent work for the duration. Added a predictive hot-range tracker (approximate LRU micro-trace) to avoid immediate re-fetching of recently evicted keys. Instrumented new histograms and counters in observability to debug scheduling decisions.

Benchmarks: Microbench with 60/40 read/write mix, 8 concurrent batchers, 256 token average sequence:
- baseline (main): p50=12.6ms, p95=68ms, p99.9=412ms
- this PR: p50=10.3ms (18% improvement), p95=56ms, p99.9=268ms (35% reduction)
Memory: steady-state resident set increased by ~3.2% under the predictive pin but peak working set remained within SLOed limits. CPU overhead for evictor workers averaged +2.4% on p50.

Testing: unit tests for scheduler state transitions, integration test simulating batch windows with randomized eviction pressure, and an end-to-end canary run on perf-canary with synthetic mixed workload for 48h.

Risks & mitigations: predictive pin could raise memory pressure — bounded pin lifetime and global pin budget prevent unbounded growth. Evictor-worker thread-safety issues identified during review were fixed by using a single-writer queue for state transitions and an atomic generation counter for batch windows. We added a new runtime flag to disable the opt-in behavior for critical deployments.

Related: follows up on prior KV work around spill coalescing and deferred eviction (see ENG-2145), but focuses on scheduling coordination with continuous batching and runtime kernel selection.
Diego Martinez: Please add more observability around decisions that delay eviction—specifically, log window_id and reason when we defer > 50ms.
Aisha Patel: Added delayed_evictions counter and a sample log line with window_id; kept log rate-limited (updated).
Karen Liu: I saw a potential race between window marker updates and eviction worker drain. Could we use a generation counter instead of time-based heuristics?
Aisha Patel: Replaced time heuristics with an atomic generation counter and adjusted worker drain logic (see d5e6f7a).
Omar Hassan: CI failed lint on selector.go; please follow kernel selection style guide and add tests for eviction_pressure hook.
Aisha Patel: Fixed formatting, added unit tests for selector bias; CI green now.
Merged by squash, commit message summarizes BatchAwareEvictor + kernel bias + predictive hot-range. No breaking changes; opt-in runtime flag defaults to enabled for new clusters.
Introduce a batching-aware KV eviction scheduler that reduces tail latency by coordinating eviction with continuous batching and biasing kernel selection under eviction pressure. Operators can tune eviction pin budgets and disable behavior via runtime flags.
