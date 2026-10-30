# Ephemeral ghost canaries and rewind-crediting for fast failovers

Motivation: Long-lived canaries add operational cost and delay rollback decisiveness. This PR introduces a lightweight 'ghost' canary primitive — an ephemeral, instrumented traffic slice that can pop in (shadow-mode) and pop out rapidly with deterministic rewind semantics. The goal is faster detection with minimal warm-up cost and automatic refunding of a 'rewind credit' budget when a ghost canary is torn down to accelerate failover and preserve SLAs.

What this PR does (high level): Adds a ghost_canary orchestrator component, a rewind-credit allocator, and integration hooks so the routing plane can promote a ghost from shadow to micro-canary with a single transactional cutover. Ghosts are intentionally short-lived (default TTL 5m) and are constrained by a per-namespace rewind credit pool. If the ghost signals a regression, routing will immediately trigger a rewind using the pre-allocated credits to roll back traffic and fast-track a larger failover.

Design notes: 1) Ghost lifecycle: create -> observe in shadow -> optionally promote -> short TTL -> auto-expire. 2) Rewind-credit: accounts tracked per-customer (namespace) and per-release; credits are consumed on promote and refunded on automatic or manual rollback depending on policy. 3) Consistency: state changes are recorded in the rollout ledger (idempotent operations) and we use the existing snapshot store for model version checkpoints so rewinds are deterministic. 4) Safety: multiple independent gating metrics (latency P99, error spikes, model drift score) are evaluated by the safety-checker before promoting. Non-deterministic regressions trigger immediate rollback without consuming additional credits.

Checklist: - [x] Core orchestrator implementation (ghost_canary.go) - [x] Rewind-credit allocator and accounting (credits.go) - [x] API for ephemeral canary config (schema + validation) - [x] Routing integration + transactional promote endpoint - [x] Integration tests simulating fast failover - [x] Docs + runbook for on-call

Testing done: unit tests for allocation logic, integration tests covering promote and rewind flows, end-to-end smoke that creates a ghost, promotes it, simulates a regression, and asserts that refunding semantics are correct. Load tests show ghost creation adds <2% overhead to routing path.

Migration notes: Adds a new optional rollout config field 'ghost_ttl' defaulting to 300s. Existing rollouts opt-in via the new field; no automatic behavior change for current canaries.

Related: ENG-1287 (fast rollback primitives), ENG-1299 (cohort budgeting).

Merge strategy: squash and merge after approval and successful CI.

Operational playbook (short): 1) When experimenting, create ghost with small cohort (0.5-1%) and set ghost_ttl to 2-10m. 2) Watch the ghost dashboard; if safety gates fail, system consumes rewind credit and rolls back automatically. 3) If ghost is healthy, can be promoted to micro-canary which consumes remaining credits for a transactional cutover.
Ethan Zhou: This is an interesting primitive. Concern: what prevents noisy signatures causing repeated promote/rewind churn?
Maya Patel (author): We added an exponential backoff for promotions and a per-namespace cooldown window tied to the credit allocator. Churn beyond the cooldown requires manual override.
Priya Malik: Can we ensure refunds are audited and appear in billing traces? Ops needs visibility into credit refunds for finance reconciliation.
Maya Patel: Credits are emitted as ledger events (credits.refund) and the billing pipeline is subscribed to these events; added unit tests to validate emission.
Luis Fernandez: Tests look good but add a deterministic replay test to prove rewinds restore exact model snapshot state.
Maya Patel: Added snapshot restore assertions to test_ghost_canary.py — now restores and validates tokenization checksum after rewind.
Introduces ephemeral 'ghost' canary primitive and rewind-credit accounting to enable fast failovers with deterministic rewinds. Ghosts are opt-in via 'ghost_ttl' and have minimal runtime overhead. See docs/rollouts/ghost-canaries.md for usage and on-call runbook.
