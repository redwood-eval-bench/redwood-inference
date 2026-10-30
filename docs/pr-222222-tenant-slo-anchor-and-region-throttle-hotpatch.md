# Tenant SLO anchor and region-aware soft-throttle hotpatch

Context: During the 2026-02-10 regional surge incident we observed cascading circuit opens in region-eu-1 when transient KV pressure + bursty tenant patterns caused a large number of tenants to be evicted simultaneously. That amplified cross-region failover and caused elevated tail latency for hosted customers. This PR introduces a short-term hotpatch that implements two coordinated mechanisms: 1) Tenant SLO anchoring: per-tenant short-term SLO credit anchoring so that a temporary regional degradation does not immediately force a tenant into global degraded routing. 2) Region-aware soft-throttle: a graded token-bucket soft-throttle at regional egress that attenuates low-priority traffic earlier and applies sticky, exponential backoff windows to avoid oscillation. The goal is to reduce global failover flaps and give operators time to steady the region without mass tenant migrations.

What this changes (single-line summary): Adds slo/anchor service + region throttle + metrics, feature flagging, and operator playbook updates.

Detailed changes: - services/slo/anchor.go: new anchored credit ledger with expiry and bias toward preserving active low-volume tenants. - pkg/circuit/throttle.go: graded soft-throttle controller that computes per-tenant allowance using regional pressure signals and tenant weight. - controlplane/flags/feature_flags.go: feature flag to enable on customers-by-tenant for staged rollout. - observability/metrics/slo_anchor_prometheus.yml: new metrics & dashboards. - cmd/ops/playbooks/region_throttle_runbook.md: operator runbook and rollback steps. - unit + integration tests and a small e2e harness that replays the incident trace for validation.

Motivation and rationale: A previous approach (immediate aggressive eviction) created a noisy global failover. Anchoring gives the system a short lease to absorb transient pressure and lets the soft-throttle shed only lower-priority work. The throttle is deliberately conservative: it signals eviction only after multi-signal confirmation (KV pressure + latency slope + queue depth) to avoid false positives.

Checklist:
- [x] Unit tests for anchoring ledger and refill semantics
- [x] Integration tests for per-tenant allowance propagation
- [x] Canary rollout on staging + eu-1 shadow traffic replay
- [x] Observability (SLO anchor hit rate, throttle activations, tenant demotions)
- [x] Ops runbook & rollback instructions

Testing notes: Canary in eu-1 for 2 hours with traffic amplification tests; synthetic tenant spike replay shows 70% reduction in simultaneous tenant demotions and 40% reduction in cross-region failovers in the harness. CI initially failed on a race in anchor expiry; addressed by commit "fix(anchor): atomic expiry updates".

Rollout plan: 1) Feature-flag on for internal customers in eu-1, monitor slo_anchor_hit_rate and demotion_count for 24h. 2) Expand to other EU regions, then NA with tiered enable. 3) Full default on after 2 weeks if metrics stable.

Risks & mitigations: The anchored credits could be abused by noisy tenants. We mitigate by: credit decay, tenant weight caps, and telemetry that triggers auto-demote for tenants exceeding burst budget. The throttle is soft and will not drop traffic for premium tiers without explicit policy.

Related: ENG-9876 (regional stability epic), incident SUP-45.
Evan Chen: Nice approach. Two asks: (1) include a tenant-level max-credit to avoid indefinite anchoring; (2) add a metric for 'anchored_recovery_latency' so we can track hidden degradations.
Author (Anika Patel): Added max-credit cap and the metric in observability commit; updated tests to validate cap behavior.
Priya Raman: I saw the initial CI race — thanks for the fix. One nit: throttle window math looks like it might allow quick reopen if only queue depth drops but latency stays high. Can you gate reopen on a moving-average of latency?
Author: Good catch — gating reopen now requires both queue depth and latency slope to be below thresholds for a 30s window. Added integration test scenario.
Luis Gomez: Approving for hotfix path. Recommend we run canary with increased telemetry retention for 48h to capture edgecases.
Hotfix: Adds tenant-level SLO anchoring and a graded per-region soft-throttle to prevent mass circuit tripping during regional spikes. Included telemetry and rollout flags.
This PR is intentionally conservative and behind a feature flag for a staged rollout. Operators: see cmd/ops/playbooks/region_throttle_runbook.md for emergency rollback (flip feature flag and restart controlplane) and metric thresholds to watch: slo_anchor_hit_rate, throttle_activation_rate, tenant_demotion_count, cross_region_failovers. Post-merge: schedule a postmortem sync with SRE and product to evaluate long-term policy adjustments.
