# adaptive-rehearsal-bridge-with-commit-latency-throttling

Context and motivation:
We need better safety when promoting canaries in workloads where commit latency (the time between accept and full commit across downstream systems) materially impacts user-facing SLOs. Existing canary shapes measure steady-state latency and error rates but often miss transient commit-latency spikes caused by downstream compaction, cache warmups, or distributed metadata churn. This PR implements a lightweight 'Rehearsal Bridge' stage that: 1) replays a sampled subset of traffic into a bounded rehearsal path that imposes realistic downstream latency constraints, 2) measures commit-latency as a first-class signal alongside p95/p99 token latency, and 3) gate-promotes based on a telemetry-led heuristic plus an automated rollback oracle if commit latency breaches tolerated bands during the rehearsal window.

High level changes (single-string summary):
- New controller: pkg/rollout/rehearsal_bridge_controller.go (implements reconciliation, phased rehearsal lifecycle, and commit-throttle enforcement)
- Rehearsal worker: internal/rehearsal/bridge_worker.go (bounded queue, token bucket enforcement to simulate throughput/latency coupling)
- Telemetry adapter: telemetry/adapters/commit_latency_adapter.go (aggregates commit-latency, ties into Prometheus summaries and event annotations)
- Rollback oracle: tools/rollback_oracle/service.go (decision engine with backoff and dry-run mode)
- Docs and runbook: docs/runbooks/rehearsal-bridge.md and examples/quickstarts/rehearsal-bridge.yaml
- Helm manifest: helm/templates/rehearsal-bridge-deployment.yaml (optional component behind feature flag)

Checklist (implemented):
- [x] Spec for commit-latency metric, measurement points, and export labels
- [x] Controller lifecycle (init -> rehearse -> hold -> commit/rollback) with safety TTL and observer sampling
- [x] Telemetry and dashboards (Grafana panel added)
- [x] Runbook for operators including abort/force-promote steps
- [x] Unit, integration and a deterministic e2e smoke test in perf-canary harness

Commits (summary):
- a1b3f4d: initial rehearsal bridge controller and API types
- b2c9e8a: add bounded bridge worker and token-bucket simulator
- c3d4f21: commit-latency adapter and Prometheus metrics
- d4e5f32: rollback-oracle decision engine, dry-run mode, and config knobs
- e5f6a43: helm chart and docs; runbook + quickstart example

Changed files (representative list; not exhaustive):
- pkg/rollout/rehearsal_bridge_controller.go
- internal/rehearsal/bridge_worker.go
- telemetry/adapters/commit_latency_adapter.go
- tools/rollback_oracle/service.go
- docs/runbooks/rehearsal-bridge.md
- examples/quickstarts/rehearsal-bridge.yaml
- helm/templates/rehearsal-bridge-deployment.yaml
- controllers/bridge_controller_test.go
- perf-canary/harness/rehearsal_integration_test.go

Small illustrative diff snippet (bridge_worker.go):
- func process(req Request) { /* old fast-path */ }
+ func process(req Request) {
+   // apply commit-latency throttle: if downstream commit window is busy
+   // pause until token available so rehearsal simulates realistic backpressure
+   <-worker.throttle.Wait()
+   actuallyProcess(req)
+ }

Review conversation (selected comments and responses):
- Evan Zhou (review): "This looks solid conceptually. Two asks: make the default component disabled behind a feature flag and ensure the telemetry labels include session_id for session-coherent gating."
- Nina Park (author): "Addressed: Added feature flag `rehearsalBridge.enabled=false` in values.yaml and added session_id label in commit adapter; updated tests to include session coherence."
- Marco Ruiz (review): "The rollback oracle uses a simple exponential backoff policy; can we add a short-circuit if p50 commit latency is stable but p99 spikes?"
- Nina Park: "Added rule: if p50 < baseline and p99 > baseline*3 for two consecutive windows, oracle moves to 'quarantine' instead of immediate rollback so ops can inspect. Documented in runbook."
- Priya Desai (security review): "Ensure the optional rehearsal worker cannot access PII; confirm it respects redaction config."
- Nina Park: "Worker reads only routing metadata and commit timestamps; added a unit test and a small redaction validation step in the adapter (telemetry/adapters/redaction_test.go)."
- Marco Ruiz (approve): "Approved after nits."
- Evan Zhou (approve): "Approved."

CI and test notes:
- Unit tests: pass (controllers + telemetry adapters)
- Integration tests: pass (rehearsal lifecycle simulated)
- e2e/perf-canary: initially failed on a flaky harness test 'PerfHarness/Replay/SlowDown' — flake traced to an unrelated quota race; reran and now passing.
- Lint: pass (golangci-lint)
- Security scan: no high findings; SAST flagged an unused import which was removed.

Operational notes and runbook highlights (docs/runbooks/rehearsal-bridge.md):
- Feature is opt-in: values.rehearsalBridge.enabled=false by default. Recommend enabling in staging for 2 weeks before production rollouts.
- Default gates: p95(latency), p99(commit_latency), error_rate, and session_coherence failure rate.
- Quarantine behavior: oracle will quarantine the rehearsal lane (stop promotions, keep shadowing) when commit-latency spike pattern detected; requires manual review to resume or operator can force-promote in emergencies.
- Fallback: if the rehearsal bridge controller loses connectivity to telemetry for >30s, it automatically fails open to 'hold' to prevent inadvertent promotions.

Backward compatibility and migration:
- No customer API changes. Controllers and worker run as optional internal components. Existing rollout CRDs gain an optional `rehearsal:` block; controllers default to ignoring this block when the feature flag is off.

Performance impact:
- When enabled, expect modest CPU increase on control-plane nodes due to telemetry aggregation (~5-8% median), and optional rehearsal workers can be scaled independently. Documented in Optimize recommendations.

Follow-ups (future PRs planned):
- (ENG-3590) refine quarantine heuristics with Bayesian changepoint detection
- Add synthetic session-level replay to better exercise session-affine commits
- Expose a UI toggle and per-rollout override in Redwood Console for manual quarantine handling

Merge outcome:
- Merged via squash on 2026-02-23 after approvals from Marco and Evan and a green CI run.
- Tags: rollout/rehearsal, safe-rollout, telemetry-gated.

If you want to test locally: see examples/quickstarts/rehearsal-bridge.yaml and run `./hack/local-rehearse.sh --sample-rate=0.05` (script included in examples). For operator troubleshooting refer to docs/runbooks/rehearsal-bridge.md (section: "Troubleshooting quorum and commit-latency anomalies").

If you have concerns about scope or want the quarantine heuristic tuned sooner, ping me (Nina) or add notes to ENG-3472.
Introduces an intermediate rehearsal bridge stage that simulates production latency and commit-throttles canaries; adds an automated rollback oracle and operator runbook. Backwards-compatible.
