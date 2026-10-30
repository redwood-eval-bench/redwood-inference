# Feedback-looped rehearsal fanout promoter with cohort-weighted quality gates

Motivation: production rollouts frequently miss emergent user-quality regressions that only appear after small cohorts exercise edge flows. This PR introduces a rehearsal-first promoter that (1) replays sampled real traffic into a rehearsal lane, (2) computes cohort-weighted quality signals, and (3) uses a feedback loop to gate promotion and orchestrate automatic rollback when cohort-level delta metrics exceed configurable thresholds.

What this PR does (high level):
- Adds a rehearsal fanout controller that can mirror live traffic into a timeboxed rehearsal window before any traffic fraction is applied to the canary. Rehearsal defaults to 30 minutes but is configurable per-route.
- Implements a cohort-weighted quality gate using a new Delta-IRR metric (Delta Interaction-Response Ratio) which mixes error-rate, semantic drift (via embeddings distance), and user abandonment signals into a single delta score per cohort.
- Implements a feedback loop that persists rehearsal-derived deltas into the rollout decision engine. If any cohort delta exceeds its configured SLO budget, the promote operation is blocked and an automated rollback escrow is created.
- Adds sampled feature-flag weighting so low-volume but high-risk cohorts (e.g., new onboarding flows) can carry amplified signal in the gating decision.
- Introduces a rehearsals dashboard slice in the Console and a short ops playbook (docs/ops/rehearsal-playbook.md) describing escalation, manual promote, and audit steps.

Why it’s different: unlike prior shadow/canary mechanisms that apply split traffic immediately, this approach enforces a dry-run rehearsal that produces replayed outputs and computes quality deltas before any real user-facing rollout. The feedback loop ties rehearsal telemetry directly to the rollout state machine, enabling deterministic auto-blocking and escrowed rollback without human ping-pong.

Checklist: 
- [x] Rehearsal fanout controller implementation (controller/rehearsal_fanout.go)
- [x] Delta-IRR metric implementation and aggregation (pkg/telemetry/delta_irr.go)
- [x] Rollout decision adapter wiring (src/rollout/decision_adapter.go)
- [x] Rehearsal harness and replay runner (deploy/test/rehearsal-harness.yaml, tests/integration/rehearsal_flow_test.go)
- [x] Console and docs updates (docs/ops/rehearsal-playbook.md, console changes)
- [x] Unit and integration tests (added ~14 new tests)

Backwards compatibility: default behavior unchanged when the rehearsal feature flag is off. The new gating fields are opt-in on the Rollout CRD.

Operational notes: the rehearsal fanout runs within a bounded resource budget and honors existing rate-limit and quota policies. Rehearsal traffic uses a special telemetry tag `rehearsal=true` so metrics and logs are isolated. The rollback escrow will keep a snapshot of successful baseline KV-cache warmers to avoid cold-start regressions during immediate revert.

Security & privacy: mirrored traffic is filtered by configurable PII redaction rules before being stored or replayed. Customer data handling follows existing private-deployment KMS policies.

Related work: builds on ENG-1748 (shadow seeding), ENG-1955 (telemetry anchors), and SUP-8421 (customer incident that motivated cohort weighting).
Rajat Venkatesh: Can you clarify how Delta-IRR handles small cohorts with single-digit requests? Suggested we add a confidence adjustment to avoid noisy early rollbacks.
Sofia Chen: Added cohort confidence scaling — small cohorts now get a Bayesian shrinkage term. See pkg/telemetry/delta_irr.go lines 78-112.
Maya Patel: How does the escrow snapshot avoid replaying stale KV cache? Request adding TTL/invalidations metadata.
Sofia Chen: Escrow now stores cache version + creation timestamp and the rollback path invalidates caches older than 5 minutes by default (configurable).
Ethan Li: Tests look good; can we add one more integration test that simulates an aborted rehearsal followed by manual promote?
Sofia Chen: Added test `rehearsal_manual_promote_test` that validates abort + manual promote path.
Introduces rehearsal fanout promoter (opt-in) with cohort-weighted quality gates (Delta-IRR). Adds console rehearsal panel and ops playbook. Opt-in Rollout CRD extension; no breaking changes when feature flag disabled.
If a blocked promote is observed with `rehearsal=true` tag: check delta_irr aggregates, ensure rehearsal window completed, validate PII redaction didn't drop critical signals. Escrow rollbacks can be triggered manually from Console if automated backout is undesirable.
Adds a rehearsal-first promotion path that replays sampled traffic, computes cohort-weighted quality deltas, and prevents risky promotes by creating an escrowed rollback when thresholds are violated. Includes Console UI and docs for operators.
