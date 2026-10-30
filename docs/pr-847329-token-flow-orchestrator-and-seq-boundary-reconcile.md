# token-flow-orchestrator-and-seq-boundary-reconciler

Context & motivation: We observe noisy p95/p99 tails in mixed-length session replays where short and long sequences interleave. Root cause analysis shows suboptimal token-level scheduling and premature KV cache evictions when boundary alignment is poor. This PR introduces two cooperating components: (1) a Token Flow Orchestrator that proactively groups in-flight requests by soft token window and preferred kernel occupancy, and (2) a Sequence Boundary Reconciler that applies lightweight heuristics to maintain primer continuity and avoid unnecessary KV cache churn.

Design summary: The orchestrator maintains a tiny, per-worker token-slot map and a scheduler that assigns arriving requests into micro-batches constrained by a token-window score (priming affinity, recent kernel selection). The reconciler runs as a short-lived pass before batching to realign primers across requests from the same logical session and to optionally stall short requests for up to a configurable ms threshold to allow better coalescing. Both features are guarded by feature flags and incremental config knobs so rollout is safe.

Commits (high-level):
- Add token_flow::orchestrator with scheduler, metrics, and tests
- Add boundary_reconciler module with primer-aware heuristics and unit tests
- Integrate orchestrator into bench-runner main path behind flag
- Add telemetry hooks: per-window occupancy, coalesce_ratio, primer_hit_rate
- Add e2e microbench scenario and new golden baseline for mixed-length sequences
- Fix race in kv_cache warmup path exposed by staging tests

Changed areas / files (high-level):
- bench-runner/src/orchestrator.rs (new)
- bench-runner/src/boundary_reconciler.rs (new)
- bench-runner/src/main.rs (hook-in, flag parsing)
- bench-runner/tests/mixed_length_coalesce.rs (new e2e)
- telemetry/metrics.toml (definitions)
- examples/mixed-length.yaml (scenario)
- docs/benchmarks/token-flow.md (design doc + knobs)

CI & checks: Full CI run initially hit a flaky test in kv_cache::warmup (see first CI run annotations). I pinned a short mutex in a follow-up commit and re-ran; final CI shows all checks green: build, unit-tests, bench-integration, lint.

Performance validation: Local microbench runs (staging cluster) show a median p95 improvement of 8-12% and p99 improvement of 7-15% for our synthetic mixed-length session workload at 32-concurrency. Memory overhead is small (~2-3% extra RSS) due to the per-worker token-slot map.

Rollout plan: Feature-gated; default off. We will enable in canary by job bench/canary-mixed-merge, monitor metrics token_flow.coalesce_ratio, primer_hit_rate, and bench baseline deltas for 48 hours before gradual ramp.

Review conversation snapshot:
- Maya Chen (reviewer): "Nice incrementalization — can you split the reconciler tests into deterministic seeds? I saw nondeterminism on my laptop."
- Sofia (author): "Good catch — added deterministic seeds to mixed_length_coalesce.rs and improved the synthetic RNG seeding; second commit added.`"
- Luca Romero: "Two nits: (1) docs file references config key `orchestrator.window_ms` but the code uses `max_window_ms`; make them consistent. (2) please add a quick comment describing the complexity of the scheduler."
- Sofia: "Addressed both nits: renamed config to `max_window_ms` and added complexity comment in orchestrator.rs."
- Priya Patel: "What is the fallback if reconciler stalls create too much latency for short interactive queries?"
- Sofia: "We expose a hard cap `stall_ms_cap` that defaults to 8ms and an absolute abort threshold; reconciler is disabled automatically for latency-sensitive routes via a per-route flag. Added note in docs and route examples."
- Maya Chen: "Approving after CI green; ensure metrics naming is stable before release."
- Maya Chen: "Approved"

Requested changes and follow-ups logged:
- ENG-4163: "Integration test that simulates skewed primer churn" (linked); small follow-up PR will land the test harness.
- Telemetry dashboard: add primer_hit_rate chart to bench-mix dashboard (tracked as ENG-4190).

Merge outcome: Squash-merged into main; commit message summarizes motivation, design, and rollout flags.

Post-merge notes: I ran an additional 12-hour soak on the canary cluster; observed no increase in OOMs and steady drop in primer churn. We left the feature off for production until the telemetry dashboard is in place.

Backward compatibility: No breaking changes; all API/runner configs are additive and behind flags.

Additional context: This PR is intended to be a pragmatic middle ground between aggressive batching (which severely increases latency for short sessions) and naive per-request scheduling. It should reduce tail variance in multi-modal workloads and serve as a building block for future kernel-aware batch placement work.

Testing & reproducibility: Reproducer instructions added to docs/benchmarks/token-flow.md; runs via `bench-runner --scenario examples/mixed-length.yaml --repeat 5000 --seed 314159`. If you need to reproduce the race that triggered the first CI failure, use seeds 7..10 which will consistently exercise kv_cache warmup paths.
Adds a new token flow orchestrator + sampling reconciler to improve sequence boundary handling in mixed-length workloads; opt-in behind flag and does not change default runner behavior.
