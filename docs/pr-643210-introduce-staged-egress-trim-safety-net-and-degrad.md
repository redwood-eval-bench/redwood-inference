# Introduce staged-egress-trim safety-net and degrade-safe-state hotfix

Summary: This hotfix introduces a staged egress-trim safety-net and a 'degrade-safe-state' transition path for regions experiencing outgoing egress saturation. Motivation: during a recent regional egress saturation incident, abrupt circuit opens and aggressive tenant eviction caused downstream cache thrash and customer-visible 5xxs. The goal here is to provide a controlled, observable step-down that (1) trims outbound egress bandwidth gradually for lower-priority routes, (2) transitions affected nodes into a degrade-safe-state with constrained ingress admission and widened timeouts, and (3) avoid hard circuit flips by adding a short probation window and metric-anchored rollback. This PR is intentionally conservative: it implements a safety-net with conservative defaults and feature gates so we can enable/disable without full rollout.

Design & changes: - egress_trim controller (src/runtime/egress_controller.rs): new staged policy that accepts a priority-weighted trim plan and applies token-bucket caps to outbound egress per-route and per-tenant. It uses the existing scheduler hooks to avoid interfering with serving fast-path. - slo_manager (src/controlplane/slo_manager.go): adds SLO-derived triggers to recommend staged trims and to enter degrade-safe-state. SLO thresholds are evaluated on a 30s rolling window and include latency and error-rate margins. - degrade-safe-state (pkg/serving/degrade_safe.go): implements a lightweight state machine for nodes to reduce admission rates, increase internal request timeouts, and enable KV-preserve fastpath. State is reversible on stable telemetry. - telemetry and tracing (observability/egress_trim_dashboards.yaml + proto changes): added new metrics: egress_trim_applied, degrade_safe_entry, degrade_safe_exit, staged_trim_progress. Also added a structured trace annotation for decisions. - tests: unit tests for staged trim scheduling and an integration harness (tests/integration/egress_trim_simulator) that simulates regional egress saturation and validates no more than 2x customer-visible error amplification compared to baseline.

Rollout plan: feature gate off by default. Canary flow: enable in single non-critical region for 1% traffic, observe metrics 6h, increase to 10% then 50% as telemetry looks stable. Hotfix mode: this branch can be toggled on globally by SRE in emergencies to avoid immediate hard-circuit behavior while a deeper fix lands.

Checklist: - [x] Design review (SRE + Runtime) - [x] Unit tests for controller logic - [x] Integration simulation validating gradual trim behavior - [x] Telemetry dashboard + alerts added - [x] Feature flagging and runtime toggle - [x] Docs (short note in runbooks) - [x] Smoke canary in nonprod region

Commits summary: - Add staged egress trim controller and policy scaffolding - Wire SLO triggers to staged trim recommendation - Implement degrade-safe-state state machine and reversible transition - Add metrics, traces, and dashboard manifest - Integration simulator and tests - Small runtime race fix (atomic state update) and test flake guard

Files changed (high level): - src/runtime/egress_controller.rs: staged-trim policy, scheduling hooks
- src/controlplane/slo_manager.go: new SLO trigger + reconciler
- pkg/serving/degrade_safe.go: state machine and admission adjustments
- observability/egress_trim_dashboards.yaml: dashboards and alert rules
- tests/integration/egress_trim_simulator/*: harness and scenarios
- proto/egress.proto: small telemetry annotations
- go.mod / Cargo.toml: bump small deps for observability helper

Why this is different from previous soft-shed approaches: prior work focused on tenant eviction and coarse circuit states. This PR prioritizes controlled egress reduction (outbound shaping) to relieve constrained egress meshes and provides a reversible degrade-safe-state that avoids full circuit open. It coordinates with SLO signals and can be used as a temporary mitigator without causing long-lived tenant evictions.
Hotfix: safe staged egress trimming to avoid abrupt circuit openings during regional egress congestion; adds degrade-safe-state transitions and extra telemetry for SLO reconciliation.
Marco Ruiz: Can you explain how the staged trim interacts with existing batchers? Risk is increased tail latency if we apply naive caps.
Priya Shah (author): Trim is applied at egress token layer just before transmission; we avoid throttling short-lived fast-path batches by exempting proven low-latency routes. Added comment in egress_controller.rs and unit test to assert low-latency exemption.
Omar Khan: Need more telemetry around rollback decisions — operators must see what triggered a degrade-safe exit.
Priya Shah: Added degrade_safe_exit metric + structured trace annotation for each decision. Also added log-level sampling to avoid log noise.
Lina Park: Found a race in the initial state flip during simulated rapid toggles (integration). Marked required change.
Priya Shah: Fixed with atomic.CompareAndSwap and added a flake-guard to the integration harness. Pushed new commit e8c9b11.
Lina Park: Approved after CI green and added smoke canary validation results to the PR summary.
