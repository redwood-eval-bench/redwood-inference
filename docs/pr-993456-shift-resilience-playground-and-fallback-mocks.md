# Shift Resilience Playground: simulator + external-dependency fallback mocks

Context and motivation: On-call drills and runbook validation are currently manual and brittle. Teams need a lightweight way to exercise runbooks and on-call playbooks against realistic dependency failure modes (partial flaps, slow degradation, intermittent 5xxs) without incurring cloud costs or affecting production. This change introduces a local 'shift-resilience' simulator and a set of lightweight HTTP/gRPC mock services that can reproduce external-dependency behaviors used in SLO breach drills.

What this PR does: Adds a CLI-driven simulator (sloctl shift-sim) that can drive: deterministic failure patterns, probabilistic degradation curves, delayed responses with long-tail latency distributions, and cascading failure scenarios. Adds a small library of mock endpoints (http-mocks/) that expose configurable fault profiles and a harness to wire them into existing runbook tests. Includes example runbook scenarios that show how responders and escalation paths would behave under simulated conditions.

Files added/changed at a glance: a new binary (cmd/sloctl/shift_sim.go), a simulations package (internal/sim/), a mock service set (tools/http-mocks/), three example runbooks (examples/runbooks/resilience-*) and CI integration in .github/workflows/sim-check.yml.

Checklist: - [x] Simulator CLI implemented and documented - [x] Mock dependency service with configurable profiles - [x] Unit tests for simulation scheduler and profile parser - [x] End-to-end integration test that runs a scenario and asserts runbook triggers - [x] CI workflow that runs quick sim smoke tests on PRs - [x] Basic docs and ’how to run a drill’ example committed

Commit summary: 1) initial simulator skeleton + CLI wiring 2) scheduler and deterministic profile engine 3) http/gRPC mock service with YAML profiles 4) example runbooks and integration tests 5) CI workflow and docs 6) address review feedback (test flakiness fixes, harden profile parsing).

Usage notes: To run a quick local drill: 1) make build && bin/sloctl shift-sim --scenario examples/runbooks/resilience-minor.yaml --mocks tools/http-mocks/config.yaml 2) Observe simulated alerts in the dev Slack webhook and see how runbook steps are executed in dry-run mode. The CLI supports --time-scale to speed up long-running degradation.

Backward compatibility and risk: This is opt-in tooling (no runtime changes to production services). The mocks are in tools/ and the CLI is install-time; nothing modifies production routing. Risk is low; primary risk was flaky tests which were remedied by deterministic RNG seeding in tests.

Related: Linear ENG-8472 (drill automation), SUP-678 (customer ask for repeatable runbook validation).

Notes for reviewers: Focus on the scheduler semantics (internal/sim/scheduler.go), the profile DSL (tools/http-mocks/profiles.md) and the integration test harness (test/e2e/shift_sim_test.go). CI pipeline runs a quick 10s scenario; longer stress scenarios are intentionally excluded from PR CI to keep runs fast.

Release notes: Added a local shift-resilience simulator and mock dependency harness to help teams validate runbooks and on-call procedures without touching production.
Marco Jensen: Could we make the time-scaling flag apply to individual stages (requested) — addressed in commit 5, added --stage-scale and updated docs.
Rosa Kim: Tests flaked on CI due to RNG; please seed or inject clock — author reply: seeded RNG and added deterministic clock abstraction; CI green now.
Devon O'Leary: Profile DSL parsing looks permissive; can we tighten validation and add schema examples? — addressed: added stricter validation, schema doc and unit tests.
Marco Jensen: +1 for approach, minor nits in docs (typo fixed)
CI Bot: initial run failed (integration timeout), rerun succeeded after test timeout bump
