# Provision-credit-decoupler-and-soft-burst-bank-for-dedicated-fleets

Motivation: tenants with bursty workloads can cause abrupt provisioning churn and cross-tenant eviction cascades when short spikes exhaust instantaneous allowances. Existing burst token mechanics are tied too closely to per-minute provisioning quotas and can amplify autoscaler oscillation.

This PR introduces two complementary concepts for Dedicated fleets: (1) a Provision Credit Decoupler that separates long-term provisioning allowance from short-term burst credits, and (2) a Soft Burst Bank that smooths transient spikes with an amortized debit/vest model. Together these reduce headroom overreaction and make the autoscaler more tolerant of short bursts without impacting long-term capacity guarantees.

What changed (high level):
- runtime/allocator: add provision_credit_ledger and soft_burst_bank modules that track per-tenant "vested credits" and "burst reserve" with configurable decay/vesting parameters.
- autoscaler: use decoupled provision allowance when calculating desired replicas; use soft-burst-bank for transient scale-up decisions and allow borrowing up to emergency_repair_ratio.
- telemetry: new metrics exported (tenant.provision_credit_balance, tenant.burst_bank_utilization, autoscaler.borrow_events_total) and histogram for borrow durations.
- API/config: fleet-level config schema and helm values for tuning vesting_delay, decay_tau_seconds, max_burst_sustain_seconds, emergency_repair_ratio.
- docs: new RFC-style doc docs/dedicated/provision-credit-and-soft-burst.md with migration steps and example tuning for real workloads.

Backward compatibility and migration: the new ledger defaults to disabled on existing clusters; enabling is a safe opt-in. A migration job can seed initial vested balances using recent 30d usage samples (implemented as an optional job: cmd/migrations/seed_provision_credits). We take care to maintain the same long-term quota accounting so accounting-based billing behavior is unchanged.

Testing and rollout plan: unit tests for ledger math and decay logic; an integration scenario harness in benchmark-lab that replays tenant spike traces; staged rollout behind a feature flag with canary fleet. Observability dashboards and alerts added to prevent runaway borrowing.

Checklist:
- [x] Core algorithm implementation
- [x] Unit tests (ledger, decay, borrow/replenish flows)
- [x] Integration replay scenario in benchmark-lab
- [x] Docs and helm values
- [x] Telemetry + Grafana panels
- [x] Migration job (opt-in)

Related: ENG-8273 (stabilize autoscaler under tenant spikes), ENG-8321 (billing-safe burst semantics), SUP-4127 (customer incident where short burst caused eviction cascade).
Marco Liu: Overall looks good. Two questions: 1) Can you show worst-case borrow latency and how it affects scaling SLO? 2) The ledger math uses float64; consider deterministic fixed-point for ledger RPCs.
Aisha Desai: Added a microbench and latency histogram in telemetry; worst-case borrow latency is <12ms in our bench. Switched the RPC wire format to int64 nanos and documented fixed-point rounding in ledger.go comments.
Sofia Patel: Please add an explicit circuit-breaker so that emergency_repair_ratio can't be abused by noisy tenants.
Aisha Desai: Added circuit-breaker: fleet circuit state that disables borrowing above a global incident threshold (configurable). Unit tests cover the circuit behavior.
Ravi Menon: Test failure in CI: flaky benchmark harness timing out occasionally. Can we increase timeout and retry in CI?
Aisha Desai: Increased replay harness timeout to 10m and added a retry wrapper for known transient failures. Reran CI.
Marco Liu: Approving after fixes. Left nits inline (naming and comments).
Introduce Provision Credit Decoupler and Soft Burst Bank for Dedicated fleets: opt-in capacity smoothing to reduce autoscaler churn during short tenant spikes. Config knobs: vesting_delay, decay_tau_seconds, max_burst_sustain_seconds, emergency_repair_ratio. Backwards compatible and disabled by default.
If enabling on production fleets, start with conservative settings (vesting_delay: 5m, decay_tau_seconds: 300, max_burst_sustain_seconds: 60, emergency_repair_ratio: 0.15). Monitor tenant.provision_credit_balance and tenant.burst_bank_utilization; alert on autoscaler.borrow_events_total > 100/min. Use migration job to seed initial balances if you want smooth bring-up.
Borrowing introduces potential denial-of-service vectors if an attacker drives sustained bursts. Mitigations: per-tenant circuit-breaker, fleet-wide emergency_repair_ratio, and billing rate-limits. We validated that none of the new APIs expose secrets and KMS interactions are unchanged.
