# fleet-guard-rail-and-adaptive-entropic-burst

Context: Customers running long-tail bursty workloads on Dedicated fleets reported periodic SLA violations when multiple tenants synchronized short high-throughput bursts. The autoscaler was reacting too late and per-tenant soft quotas allowed excessive instantaneous consumption. This PR introduces a Fleet Guard Rail: an adaptive, history-weighted entropic burst containment policy paired with a lightweight utilization heartbeat aggregator. It smooths transient peaks without hard throttles and produces stable signals for the autoscaler and billing pipelines.

What this PR does (summary):
- Introduces a new controller (fleet/guardrail) that computes per-tenant entropic burst scores using a decayed history window and a probabilistic allowance function.
- Adds a utilization-heartbeat service that publishes 5/50/95 latency and utilization quantiles and an occupancy fingerprint every 10s to the telemetry bus.
- Integrates guardrail outputs into the autoscaler input pipeline so the scale decision factor is weighted by burst-risk (soft multiplier in range [0.6,1.0]).
- Adds an emergency cap path that triggers when a tenant exceeds hard safety thresholds for sustained >3s overload; this emits alerts and gracefully sheds best-effort requests while preserving latency SLOs for committed capacity.
- Adds unit and integration tests for the new allowance curve and heartbeat serialization; adds canary deployment rollout support for the guardrail feature-flag.

Design notes:
- Entropic burst score: uses exponential decay (alpha=0.15 default) over the last 10 minutes of observed instantaneous QPS and token/samples; converts to a 0..1 score via a logistic transform to avoid extreme sensitivity. The allowance is sampled probabilistically (Bernoulli) to avoid synchronized hard drops for large fleets.
- Heartbeat payload is intentionally compact (timestamp, pool_id, tenant_id, occup_pct, util_q5, util_q50, util_q95, entropy_score_bucket) to keep telemetry ingestion cost low.
- Autoscaler integration is additive: final_scale_factor = base_scale * clamp(1 - gamma * burst_risk, 0.6, 1.0) where gamma is configurable per-pool. This preserves predictable minimum throughput while biasing against allocating extra warm capacity to high-risk tenants.

Files changed (high-level):
- runtime/fleet/guardrail_controller.go: new controller with allowance algorithm and emergency-cap logic.
- runtime/autoscaler/input_bridge.go: accept guardrail signals and apply soft multiplier.
- telemetry/heartbeat/aggregator.go: polling + compact payload encoder + kafka producer.
- telemetry/schema/v1/heartbeat.proto: new schema (small, additive).
- ops/featureflags.yaml: feature flag and canary rollout config.
- integration-tests/fleet/guardrail_test.go: synthetic burst scenarios + assertions.
- docs/ops/guardrail.md: operator guide and rollback steps.

Commits: See below for a concise commit list included in this PR body.

Testing and validation:
- Unit tests: coverage for allowance curve math, sampling, and heartbeat encoding.
- Integration tests: local cluster test simulating 500 tenants with mixed burst profiles; validates that 95th latency for committed tenants stays within SLO and overall tail errors decrease by ~22% in synthetic repro.
- Performance: microbenchmarks for guardrail controller loop show ~2.1us average per-tenant evaluation on 2.5M tenant/sec simulated scale (memory and CPU within budget).
- Staging canary: rolled to 5% of fleets with feature flag, observed no regressions to autoscaler throughput and a 14% reduction in emergency-scale events during controlled traffic injection.

Rollout plan:
1) Deploy to staging and run canary harness (already done).
2) Enable gate for 5% of fleets for 48h and monitor telemetry: occupancy, burst_score, emergency_cap events.
3) Gradually increase to 25% -> 50% -> 100% over 7 days if telemetry stable.
4) Full rollout triggered by oncall signoff and automated regression absence.

Rollback:
- Feature flag gate allows immediate disable; autoscaler disregards guardrail signals when gated.
- If disabled, we will revert the bridge patch and re-run a targeted canary.

Related work: Linked to ENG-2031 (autoscaler inputs refactor) and ENG-2072 (telemetry compact schema). Related support ticket SUP-4321 reported customer SLA hits motivating this work.

Checklist:
- [x] Unit tests added
- [x] Integration tests added
- [x] Docs (ops) updated with runbook and rollback
- [x] Feature flag + canary config added
- [x] Performance review completed

Notes for reviewers: focus on safety boundaries in guardrail_controller.go and the probabilistic sampling choices. I left gamma and decay alpha as cluster-level config with conservative defaults; happy to tune with ops after initial telemetry backs them.
David Chen (reviewer): Can you walk through why we chose a probabilistic allowance vs. deterministic rate-limit? Concerned about reproducibility in debugging.
Marina Lopez (author): The sampling reduces synchronized truncation across many fleet nodes; we log sampled events and the score trace so debugging is deterministic when reproducing a tenant (score + seed logged). Added a comment and metrified sampled-droppings.
Priya Nair (reviewer): The emergency cap looks good; please ensure alerts include tenant+pool and last 3 heartbeat samples. Also add a flag to raise the hard threshold in emergencies.
Marina Lopez (author): Added extra fields to alert payload and an ops-configurable override for emergency threshold. Updated docs with the alert format.
Ethan Park (reviewer): Integration tests flaked once on CI (timing) — can we make them less timing-sensitive?
Marina Lopez (author): Tightened mocks and converted a timing assert to eventual consistency with 30s window; re-ran CI and tests stabilized.
David Chen (approver): Code looks good; leaving a minor nit about naming in input_bridge (suggest rename "burstRisk"->"burst_penalty").
Marina Lopez (author): Renamed variable and pushed. Thanks!
CI: flaky/test-autoscaler-timeouts failed on first run, rerun passed. All checks green.
Introduce Fleet Guard Rail: adaptive entropic burst containment, utilization heartbeat telemetry, autoscaler integration. Improves latency stability for bursty tenants and reduces emergency-scale events. Config flags available for staged rollout.
Post-merge: ops will enable 5% canary on 2026-03-01 and monitor occupancy and emergency_cap metrics for 72h. Follow-up item to surface per-tenant burst_risk in the Console (ENG-2109).
