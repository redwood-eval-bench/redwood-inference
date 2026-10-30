# Intervalized Reservation Ledger and Credit Reclaiming for Dedicated Fleet

Context and motivation: Dedicated fleets currently support per-tenant reservation credits and a burst token pool. In practice we've seen long-lived unused reservations and noisy signals that cause persistent warm pools and elevated cost. This PR introduces an intervalized reservation ledger: reservations are represented as deposits that decay/settle on finite intervals (configurable, default 1m).

Goals: reduce long-tail reserved capacity, make burst windows explicit for accounting, enable safe reclaim of stale credits, and expose interval-level utilization telemetry so autoscalers can reason about short-term vs sustained load.

Design summary: The ledger stores credits keyed by tenant+reservation-id with an interval epoch (iso8601 minute bucket). New reservation deposits are stamped to the current epoch. Consumption (allocation) subtracts from the active epoch first, then older epochs if allowed by policy. A background reconciler periodically (default every 30s) performs reclaiming: it marks epochs older than a configurable grace window as reclaimable and emits audit events. Reclaim is soft by default (marks credits as 'draining' and prevents new allocations from using them) and is switchable to hard reclaim (instant withdraw) for aggressive cost-saving.

API changes and config knobs:
- new config: fleet.reservations.interval_seconds (default 60)
- new config: fleet.reservations.grace_intervals (default 3)
- new config: fleet.reservations.soft_reclaim=true/false
- admin endpoint: POST /admin/ledger/force-reclaim?tenant={id}&epochs=2 (RBAC-protected)
- exported metrics: redwood_fleet_reservation_epoch_count{tenant,epoch_status}, redwood_fleet_reservation_reclaimed_total{tenant,method}

Implementation notes: core logic implemented in pkg/fleet/ledger.go and an IntervalizedAllocator wrapper in pkg/fleet/allocator/intervalized_allocator.go. The reconciler runs as a bounded goroutine with a name and backoff; ledger updates use a CAS loop on the reservations KV to ensure correctness under concurrent allocators. We added an in-memory mock to the existing lease simulator for unit/integration tests.

Safety and rollout: default behavior is backward compatible (interval_seconds set to 60, grace_intervals high enough that no immediate reclaim occurs). Feature gated behind feature flag "intervalized_reservations" configurable per-cluster. Rollout plan: ship as opt-in (console flag + operator config), collect telemetry for 2 weeks, then enable conservative reclaiming for low-risk tenants.

Migration: existing reservations are migrated to an initial epoch equal to the current server minute on first reconcile. No migration downtime expected.

Testing: added unit tests for ledger arithmetic, CAS correctness, and TTL/reclaim transitions. Integration test simulates burst traffic and asserts reclaimed credits are not used after drain. Performance microbench shows ~3% CPU overhead and sub-ms median allocation latency for common fast-paths.

Related: ENG-3421 (fleet reservation decay prototype), ENG-3498 (reclaim telemetry unwind).
Adds an intervalized reservation ledger that credits and reclaims tenant reservations on fixed windows to improve burst fairness and reduce long-lived overprovisioning. Exposes per-interval utilization metrics and soft reclaim policies.
Nina Patel (review): Can we guarantee atomicity when two allocators reduce the same epoch concurrently? The CAS loop looks okay but add a comment and a unit test for the race case.
Avery Chen (reply): Added a unit test that simulates concurrent allocators and documented the CAS loop in ledger.go (see commit d2c9e54).
Diego Morales (review): Telemetry cardinality looks high if we label by epoch timestamp per-tenant. Can we bucket epochs in retention or expose only current/older/total?
Avery Chen (reply): Changed metrics to expose epoch_status label (current/draining/reclaimed) and emit epoch identifiers only to the audit log. This keeps Prometheus cardinality bounded.
Harper Liu (review): What is the operator upgrade path? If an operator restarts mid-epoch will we duplicate reclamation?
Avery Chen (reply): Reconciler is idempotent; we mark reclaimed epochs via an idempotent state transition and use lease ownership to ensure only one reconciler per cluster runs reclaim. Added integration test to simulate restarts.
