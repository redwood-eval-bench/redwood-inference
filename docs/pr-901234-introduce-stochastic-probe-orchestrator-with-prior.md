# Introduce stochastic probe orchestrator with priority fairness and jittered scheduling

Context and motivation: Canary probes are an important signal for continuous performance monitoring, but in high-burst or highly-multiplexed traffic patterns we observed probe collisions, route starvation, and noisy signals that cause false-positive rollbacks. Previous strategies (fixed-rate probes, static jitter per-route) failed to balance probe coverage and stability when traffic shapes change rapidly.

Proposal: This PR implements a stochastic probe orchestrator that schedules lightweight mini-canary probes using a combination of: prioritized token-buckets per route, randomized jitter sampling (exponential jitter windowing), and a probe backoff controller that adapts intensity based on recent probe utility and local SLO pressure. The orchestrator is pluggable into the existing canary evaluator and exposes a small, typed config surface.

Key features implemented:
- Priority fairness: probes are assigned priority classes (critical, standard, opportunistic). The orchestrator enforces fairness by using per-priority quotas and windowed deficit counters so low-priority routes are not starved during bursts.
- Stochastic scheduling: instead of fixed intervals, probes are scheduled with jitter drawn from an exponential distribution scaled by route load; this reduces synchronized collisions across many routes.
- Adaptive backoff and intensity scaling: probe intensity decays on noisy or low-utility probes and increases when the evaluator requests more samples during suspected regressions.
- Integration with rollout guards: probes now feed a new probe_stability signal to the rollout guard which can bias pause/rollback thresholds when probe starvation or collision is detected.
- Observability: emit probe.scheduled, probe.attempt, probe.success, probe.latency.hist (p50/p95/p99), probe.starvation_rate, fairness_score (windowed). Dashboard panels and alert rules added in observability-pack.

Checklist:
- [x] Core orchestrator implementation (scheduler, quotas, backoff).
- [x] Integrate with existing canary evaluator interface and expose hook for rollout guard.
- [x] Unit tests for scheduler fairness and jitter distribution edge cases.
- [x] Integration tests simulating bursty traffic and multi-tenant probe contention.
- [x] Metrics and Grafana dashboard additions.
- [x] Docs: configuration reference and tuning guide.

Commit summary:
- feat(orchestrator): add stochastic probe scheduler and priority quota subsystem (initial impl)
- fix(tests): address race in probe lifecycle unit tests by adding deterministic clocks
- feat(observability): add probe metrics and Grafana panels; update alert rules
- perf(sim): add synthetic bursty traffic harness used in integration tests
- docs: add operator tuning guide and examples of fairness configs

Files changed (high-level):
- pkg/canary/orchestrator/scheduler.go (new) 
- pkg/canary/orchestrator/quota.go (new)
- pkg/canary/orchestrator/backoff.go (new)
- pkg/canary/orchestrator/config.go (new config types + validation)
- pkg/canary/evaluator/bridge.go (hook added to consume probe signals)
- test/integration/burst_simulator.go (new harness)
- test/unit/scheduler_test.go (fairness/unit tests)
- observability/grafana/probe_dashboards.json (panels added)
- docs/operators/probe_orchestrator.md (tuning guide)
- pkg/rollout/guard/probe_integration.go (rollout guard integration)

Review conversation (high level):
- Miguel Ortega (reviewer): "Nice approach — priority quotas look promising. Two asks: 1) add a deterministic test for the exponential jitter sampler edge case, 2) can we expose a single boolean toggle to opt out for older private deployments?"
  - Author reply: "Added a deterministic sampler in tests and added a simple feature flag (orchestrator.enabled) with a deprecated note in the config validation. See second commit."
- Lena Fischer (reviewer): "Please add metrics for probe collision rate and a short description in docs about how operators should interpret fairness_score."
  - Author reply: "Added probe.collision_rate and expanded docs with a short operator FAQ and recommended alert thresholds."
- Ravi Menon (reviewer): "Integration tests look flakey in CI under resource pressure. Can we make the simulator less CPU-hungry and rely on wall-clock pacing?"
  - Author reply: "Adjusted the harness to use time.Ticker pacing; fixed the flakiness and reran integration tests."

CI summary:
- Build: pass
- Unit tests: pass (deterministic scheduler tests added)
- Integration tests: flaky on first run (timeout in simulator), rerun -> pass
- Lint: pass

Observations from integration run: under a simulated 3x burst the orchestrator reduced probe collisions by ~72% and maintained >95% fairness_score for standard routes while preserving critical-priority probe coverage. Probe-originated false positives in the canary evaluator dropped by ~40% in the synthetic workload.

Backward compatibility and config: The orchestrator is opt-in via config: "orchestrator.enabled=false" by default for older Dedicated/Private customers; the feature flag is documented. Config schema includes per-priority base rates and a global max_probes_per_window.

Follow-ups and TODOs:
- Add a lightweight canary-aware admission controller that coordinates probe rates across multiple instances in Dedicated clusters (ENG-1023).
- Evaluate finer-grained fairness algorithms (weighted deficit round robin) against token-bucket quotas in production pilot.
- Add a small UI in Redwood Console to visualize live probe scheduling and per-route quotas.

Merge notes: Squash-merged after approvals. Patch-level rollout: deployed to perf-canary staging for one week before promoting to prod. Monitoring dashboards created and alerts set to notify SRE if probe.starvation_rate > 0.1 for >5m.

Related work and references: ENG-987 (canary robustness), PM-102 (operator tuning), prior PRs on probe scheduling and burst shielding.

If you want to reproduce locally: run test/integration/burst_simulator.go with ORCHESTRATOR_ENABLED=true and use the provided small-sample config in docs/operators/probe_orchestrator.md.
- Add a stochastic probe orchestrator that reduces probe collisions during bursts.
- Introduce priority classes for probes to ensure fairness across tenant routes.
- New observability metrics (probe.scheduled, probe.starvation_rate, fairness_score).
- Rollout guard integration: adaptive probe intensity with automatic cooldown on regression detection.
