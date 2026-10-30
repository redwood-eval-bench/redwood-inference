# Mini-brokered canary with adaptive shadow throttling and heartbeat resume

Motivation: Small-scale canaries (mini canaries) are widely used across services to validate model swaps without the overhead of a full canary lane. Current controllers either treat mini canaries like full canaries (high cost) or rely on coarse shadow mirroring (no fast rollback signal). This PR introduces a 'brokered' mini-canary mode that mediates canary traffic through a lightweight broker, applies adaptive shadow throttling based on signal noise and capacity, and supports heartbeat-resume to safely continue paused promotions.

What this PR does: Adds a CanaryBroker runtime component, a small in-cluster control loop that maintains leases for mini canaries, tracks heartbeat pings from canary workers, and coordinates incremental traffic handoffs. Shadow traffic is routed through the broker and throttled by an adaptive policy (decays throttle when variance is high). Heartbeat-resume allows paused canaries to resume after operator-acknowledged quiesce windows; automatic resume only occurs when telemetry gates are green for 5 consecutive minutes. Includes a safety playbook that triggers auto-revert if error-rate or latency regressions cross configured thresholds for the canary window.

Design notes: - Broker leases are ephemeral and scoped to cluster/namespace; no cross-tenant state. - Adaptive throttling uses an exponential moving variance estimator over 60s windows; throttles shadow traffic to maintain stable sampling budgets. - Heartbeat is a 1s ping from canary worker; missing 3 consecutive heartbeats will pause the canary and drain canary fraction gradually. - Telemetry gates are evaluated using existing SLO metrics (p95 latency, error_rate, and a custom quality metric) with configurable tolerances.

Commits summary: - c1a2f3d: controller: add CanaryBroker skeleton and lease controller - b2d4e6f: routing: brokered route plumbing + shadow throttle hook - d3e5f7a: telemetry: gate evaluator + unit tests - a4b6c8e: cli: rollout CLI flags for brokered mode - e5f7g9h: docs: canary playbook and examples

Files changed (high-level): - controller/pkg/broker/*: new broker loop and lease manager - routing/handlers.go: brokered route integration and throttle hook - telemetry/gates.go: gate evaluator + mocks - helm/charts/redwood/templates/rollout-crd.yaml: new config fields - docs/runbooks/mini-brokered-canary.md: operator playbook - tests/*: end-to-end for mini canary lifecycle

Review conversation (high level): - Daniel Park (reviewer): "This looks solid; my main concern is lease churn under heavy churn. Can we backoff renewals?" - Author reply: "Added jittered exponential backoff for renewals; added unit test for backoff behavior." - Amina Lopez: "Suggest making the default throttle conservative (10%) until teams opt-in." - Author reply: "Default set to 5% and documented. Added feature flag canary.brokered.defaultThrottle." - Marco Stein: "Telemetry gating logic needs an audit trail for post-mortem." - Author reply: "Added trace events and audit log entries for gate evaluations; they appear in console under rollout-traces.

CI summary: - unit-tests: pass - integration-e2e (k8s:1.27): pass (flaky test retried once) - lint: pass - benchmark-smoke: pass - canary-simulation: pass

Validation: End-to-end smoke run against staging cluster demonstrated a mini canary promoting 2% -> 10% -> 30% with shadow at 50% throttled down to 10% dynamically; heartbeat pause/resume exercised by killing canary worker and restoring it; auto-revert triggered correctly when injected latency jump exceeded threshold.

Backward compatibility and rollout: Backwards compatible; brokered mode is opt-in via YAML flag canary.brokered.enabled=false by default. No changes to existing rollout CRDs unless flag enabled. Release notes in CHANGELOG.

Merge: squash-and-merge performed after CI green and 2 approvals. Merged by Priya Nandakumar on 2026-02-02.

Related: Linked Linear tasks ENG-9052 (mini-brokered canary design) and ENG-9170 (telemetry gates).
Introduces a mini-brokered canary controller that enables low-latency canary routes with adaptive shadow throttling; adds heartbeat resume to automatically recover paused canaries. Backwards compatible. Config flags under 'canary.brokered' in rollout CRD.
