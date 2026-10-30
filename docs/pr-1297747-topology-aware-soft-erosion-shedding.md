# Introduce topology-aware soft-erosion tenant shedding and quarantine policy

Summary: Adds a topology-aware "soft erosion" shedding policy and tenant quarantine stage for transient regional distress. The goal is to reduce SLO violations during noisy spikes by gently degrading non-critical tenants nearest to a stressed availability zone before escalating to full circuit states. This is an operational-first, low-risk mechanism designed to be enabled via runtime flags and rolled out to canary regions.

Motivation and context:
- Recent incidents (see linear ENG-3248) showed rapid regional tails where heuristic load-shedding tripped broad tenant circuits causing noisy failovers and cascading cross-region traffic.
- We need an intermediate "erosion" phase that performs: (1) topology-aware preference trimming (prefer local traffic to stay on healthy sub-zones), (2) low-impact tenant soft-shedding that reduces concurrency or token budget for low-priority tenants, and (3) a short-lived quarantine timer to observe whether a tenant's traffic stabilizes before marking circuits.

Design overview:
- New shedding strategy: TopologySoftErosion. It evaluates health per availability sub-zone and a tenant score derived from SLA tier + recent error/latency trend. Instead of immediate circuit flip, the runtime applies a proportional reduction in concurrency and a token-budget cap per request category. Reduction is smooth (exponential decay over a configurable window) so callers see progressive latency increase rather than abrupt failures.
- Quarantine stage: when a tenant crosses the configured erosion threshold three times within a window, we move it into a short quarantine with an extended backoff evaluation. Quarantine is tenant-scoped only and does not trigger global capacity reassignments.
- Topology hints: scheduler exposes sub-zone load and a bias term that allows the soft-erosion controller to prioritize shedding tenants that will reduce cross-sub-zone tail amplification.

Runtime flags and rollout plan:
- Feature flag: runtime.experimental.topology_soft_erosion = true (off by default).
- Canary plan: enable in us-west-2a only for 10% of traffic for 48 hours, then progressively widen.
- Fallback: any unexpected error in the controller yields an immediate no-op and full pass-through. Configurable kill-switch for SRE.

Files changed (high level):
- src/runtime/shedder/topology_erosion_controller.go -- main controller and policy logic
- src/runtime/shedder/score/tenant_score.go -- tenant scoring heuristics and unit tests
- src/scheduler/topology_bias.go -- exposes sub-zone bias API to shedder
- pkg/telemetry/erosion_metrics.go -- new metrics (erosion_applied, quarantine_enter, quarantine_exit, tenant_shed_fraction)
- deploy/flags/runtime_flags.yaml -- new flags and defaults
- internal/e2e/heavy-load-soft-erosion.test -- integration test to validate progressive degradation
- docs/operations/soft_erosion.md -- runbook and SRE playbook

Small illustrative snippet (from topology_erosion_controller.go):

if tenantScore < cfg.ErosionThreshold {
    // proportional concurrency scale-down (0.0..1.0)
    scale := math.Exp(-float64(cfg.DecayRate) * windowNormalized)
    targetSlots := int(math.Max(1, float64(currentSlots) * scale))
    runtime.ThrottleTenant(tenantID, targetSlots)
    metrics.Inc("erosion_applied")
}

(keeps changes deliberately small and focused on the controller; most other touched files wire metrics and flags)

Testing and validation:
- Unit tests for tenant scoring heuristics (tenant_score_test.go).
- Integration e2e that injects synthetic sub-zone latency skew and verifies that low-priority tenants degrade gracefully while SLO-critical tenants maintain their allocation.
- Load test in benchmark-lab that measures token cost delta and tail latency before/after.

Review notes and trade-offs:
- We intentionally avoid hard OOM or eviction decisions inside soft-erosion; it's a priority-reduction layer only. True circuit-break semantics remain unchanged.
- The risk is mis-scoring tenants; to mitigate we added conservative defaults and a canary rollout path.
- This PR adds telemetry hooks to allow rapid rollback if unintended customer impact is observed.

Checklist (this PR):
- [x] Implementation of controller and scoring
- [x] Unit tests and basic integration tests
- [x] Telemetry + runbook draft
- [x] Feature flags and kill-switch
- [x] Canary rollout plan in PR description

Operation and rollback:
- To disable immediately: set runtime.experimental.topology_soft_erosion=false and flush controller caches.
- For partial rollback, set canary percent to 0 for the feature flag.

Related work: ENG-3311 (regional traffic stabilization), several prior PRs that added tenant prioritizers are complementary.

Merge strategy: squash merge, will tag release with canary channel.

Notes for release: monitor metrics: erosion_applied, quarantine_enter, quarantine_exit, per-tenant latency and token cost, and a new alert "SRE/soft-erosion-unexpected-spike" which will fire if quarantine_enter rate exceeds threshold in a region.
Marco Rivera: Please add a safety bound to ensure we don't reduce slots below 1 — otherwise a tenant could be fully starved during pathological scoring.
Asha Patel (reply): Added a floor (min 1 slot) and included unit test. See tenant_score.go and controller comments.
Eleni Katsaros: Can we add an explicit metric that distinguishes canary erosion vs global erosion so we can filter noise?
Asha Patel (reply): Added erosion_scope label on erosion_applied metric. Updated telemetry accordingly.
Samir Bhandari: Integration test looks good; please run benchmark-lab scenario before merging. Also confirm SLO dashboards are not noisy in canary.
Asha Patel (reply): Ran benchmark-lab profile and attached summary to the PR. Tail 99.9% improved for critical tenants by 8–12% under simulated skew.
Introduces a topology-aware soft shedding policy and tenant quarantine stage to reduce regional tail amplification. Feature gated behind runtime.experimental.topology_soft_erosion; canary rollout recommended. Adds new telemetry metrics and an SRE runbook.
