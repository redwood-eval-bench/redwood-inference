# adaptive-tenant-throttle-reconciler-with-conditional-lease-fallback

Motivation: recent cross-region probe flaps caused noisy tenant-level throttle flips during short-lived control-plane hiccups (see SUP-9876). This PR implements an adaptive tenant throttle reconciler that smooths probe inputs, enforces SLO-aligned hysteresis, and provides a conditional lease fallback path to a standby serving tier when a tenant's primary region is unstable. The goals are: reduce rapid tenant preemptions, maintain latency SLO for high-tier customers, and provide a fast, auditable recovery path for transient control-plane probe failures.

What this changes (high level): adds a new reconciler service (reconciler/tenant_reconciler) that subscribes to health probes and tenant metrics, computes a stable lease state with exponential smoothing + min-stable-window, and issues conditional leases to the runtime scheduler. When a tenant's primary lease is revoked due to sustained instability the reconciler will offer a conditional fallback lease to a standby region/model variant instead of immediate eviction. The runtime scheduler receives a lease API (runtime/lease_manager.go) to accept/reject conditional leases and exposes a decision reason for observability.

Key behavior and knobs: single config flag enable_tenant_reconciler (default: false); smoothing_alpha (0.2); min_stable_window_ms (15000); fallback_grace_ms (120000); fallback_policy: [prefer-local-standby, degrade-quantize]. Reconciler emits metrics: reconciler.lease_decisions, reconciler.signal_smoothed, reconciler.fallback_offers.

Files changed include a small API surface on the scheduler and about six new modules for reconciler logic and tests. No model serving semantics were changed. Backwards compatibility: reconciler is gated by a feature flag and defaults to off. Rollout plan: feature-flag canary in eu-west-1 at 5% traffic for 24h, then 25%, then full rollout. Rollback: flip enable_tenant_reconciler=false and revoke conditional leases via admin endpoint.

Testing: unit tests for smoothing logic, integration test harness that simulates probe jitter and asserts stable decisions, and a small load canary run (k8s canary job) validated against latency SLOs. CI: lint, unit, integration (mock), end-to-end canary job.

Related: ENG-2345, ENG-2419, SUP-9876. Linear links and monitoring runbooks updated in internal doc ID: linear/ENG-2345.

Checklist:
- [x] Design doc (linked in PR)
- [x] Unit + integration tests
- [x] Canary plan and rollback steps
- [x] Observability metrics and dashboards wired

Why this is different from previous approaches: instead of immediate tenant eviction this adds a temporal reconciliation step that prefers conditional fallback offers and gives the system time to absorb short probe regressions without noisy tenant churn. This reduces churn-induced latency spikes and decreases the number of urgent incident pages for probe flaps.
Lena Park (reviewer): Can we add a metric for 'fallback_offer_accept_rate' so we can see how often standby accepts are used? Also please document the expected cost delta for fallbacks.
Dmitri Volkov (author): Added metric reconciler.fallback_accept_rate and included cost note in runbook (docs/runbooks/tenant-reconciler.md).
Sameer Kapoor (reviewer): The min_stable_window of 15s seems conservative for small tenants; suggest making it configurable per-tenant tier. Consider adding tier_multiplier in config.
Dmitri Volkov (author): Exposed tier_multiplier in config and updated tests to cover high/low multiplier scenarios.
Marta Ruiz (approver): Verified canary telemetry (eu-west-1, 5%) for 24h. No increase in P95 latencies for gold tenants. Approving.
Introduce tenant-level reconciler to smooth probe signals and provide conditional fallback leases to standby regions. Feature gated; enabled via feature flag and rolled out by canary. Improves stability during short-lived control-plane probe flaps; reduces noisy tenant evictions.
Start canary in eu-west-1 at 5% traffic for 24h, monitor reconciler.lease_decisions and runtime.latency_p95 by tenant tier. Increase to 25% then full after 72h of stable telemetry. Rollback by disabling enable_tenant_reconciler flag and invoking admin revoke endpoint.
