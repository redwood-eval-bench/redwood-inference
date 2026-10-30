# policy-sentinel-rollout-scheduler: cohort draining and sentinel hook integration

Motivation: We need a lightweight, policy-driven scheduler to protect user traffic during staged rollouts while giving platform teams fine-grained hooks to quarantine problematic cohorts. This PR introduces a Policy Sentinel Scheduler that coordinates cohort draining, injects short-lived sentinel probes, and enforces per-cohort safety gates (error-rate, latency, and semantic-regression probes).

What this PR does: 1) New scheduler component (pkg/controller/scheduler/sentinel_scheduler.go) that evaluates rollout policies and generates drain/hold/quarantine actions; 2) Per-cohort sentinel hooks: a configurable sequence of probes that run in shadow mode and report structured signals to the rollout engine; 3) Quarantine window semantics: cohorts entering quarantine are automatically held and optionally replayed to a verifier service; 4) Integration with existing rollout CRD (api/rollouts/v1beta1.proto) via optional policy.sentinel block; 5) A small telemetry shim that emits sentinel events to Console (console.events.rollout.sentinel) for observability and SRE alerting.

Implementation notes: The scheduler operates as a controller with a pluggable policy evaluator. Policies are expressed as YAML mappings with thresholds and sequences (e.g., latency_p50 < 150ms, semantic_mismatch < 0.7). Sentinels are small ephemeral jobs that can be configured to run either as shadow traffic (no latency SLO impact) or as validation only. To avoid noisy alerts during legitimate warmup, the scheduler supports a 'stability grace' period and backoff escalation for repeated flaps.

Testing and rollout plan: Unit tests cover policy parsing and decision logic. Integration tests (tests/integration/rollout_sentinel_test.go) simulate cohort-state transitions including success, transient error bursts, and persistent semantic regressions. CI added a deterministic random seed to make the integration stable. Rollout will be gated behind a feature flag; initial rollout to staging uses percentage-based cohort assignment with two sentinels (health-check, semantic-eval). Observe console.events and the new per-route sentinel dashboard before enabling in production.

Backwards compatibility: Existing rollouts without policy.sentinel remain unchanged. The new scheduler only activates when the policy is present or the cluster admin enables rollout.policy.sentinel.enabled via ClusterConfig.

Security and perf: Sentinels run with minimal RBAC - read-only where possible - and are rate-limited. The scheduler is designed not to add per-request latency because sentinel probes are asynchronous shadow requests.

Related: ENG-9823 (policy primitives), PM-1102 (product acceptance criteria), SUP-4217 (customer incident that motivated quarantine semantics).
Arjun Patel: Can you clarify the failure mode when the verifier service is down?
Maya Calder: If the verifier is unavailable the scheduler marks the cohort as 'verification-unavailable' and either holds traffic or routes to fallback based on policy.allowed_on_verifier_down. Default is hold. Added unit test for this.

Helena Wu: The integration test intermittently failed in CI due to timeouts — can we avoid reliance on wall-clock sleeps?
Maya Calder: Replaced sleeps with synthetic clock in tests and added deterministic seed to the async probe runner. CI rerun passed.

Diego Morales: Small nit - event names are verbose. Could we shorten them?
Maya Calder: Kept verbose names for initial release for clarity; added mapping in console to display compact aliases.

Arjun Patel: Approve after the race fix and docs expanded with steps to toggle the feature flag.
Helena Wu: Approved.
Diego Morales: Approved and pushed squash/merge once green.
Adds a policy-driven sentinel scheduler that enables cohort-aware draining, per-cohort safety sentinels, and a quarantine window for failed probes. Backwards compatible with existing rollout configs; opt-in via rollout.policy.sentinel.enabled=true.
Opt-in feature behind rollout.policy.sentinel.enabled. Operators should review docs/runbooks/rollout-sentinels.md before enabling in production. Suggested initial config included in docs/examples/.
