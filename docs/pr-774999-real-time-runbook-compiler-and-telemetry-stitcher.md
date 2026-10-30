# Real-time Runbook Compiler and Telemetry Stitcher

Context and motivation: On-call responders frequently have to assemble fragmented signals (alerts, traces, logs, SLO deltas) into an actionable runbook under time pressure. This PR introduces a Runbook Compiler service that composes a compact, prioritized runbook from a set of incident signals and a Telemetry Stitcher that pulls and correlates relevant telemetry slices across services. The goal is to reduce TTR and make playbook suggestions reproducible and auditable.

What this PR does (summary):
- Add runbook-compiler microservice (go) that accepts an incident payload and returns a prioritized runbook (steps + rationale + safety gates).
- Add telemetry-stitcher worker (python) that builds correlated telemetry slices (spans, top traces, key logs, metric cutouts) and applies automatic sanitization rules to strip PII/keys.
- Add ephemeral-debug-proxy (node) that can safely replay a limited subset of traffic against replica services in a sandboxed environment. The proxy enforces resource/time budgets and kill switches.
- Integrations with slo-toolkit sampler: sample sets are used to pick representative traces for evaluation and to detect regressions compared with baseline runs.
- Console + incident-bot UI hooks: new /runbook-compile command and a RunbookDraft view in incident context.

Design notes and safety:
- Sanitization: telemetry-stitcher runs a tag-based sanitizer followed by deterministic hashing for identifiers. We maintain a whitelist/blacklist config stored in vault. Changes to sanitization config require operator approval.
- Access control: runbook compilation is gated to oncall+incident-bot roles; ephemeral proxy ACLs are short-lived, minted per-session via the existing token broker.
- Resource constraints: debug proxy and stitcher are constrained by CPU/memory/QPS budgets and will fallback to a degraded mode that returns pointers to raw telemetry if budgets are exhausted.

Commits (high-level):
- 1a2f3d4 add runbook-compiler service + core compilation pipeline
- 2b3e4f5 implement telemetry-stitcher and sanitization rules
- 3c4d5e6 ephemeral-debug-proxy + sandbox harness with safe rollback hooks
- 4d5e6f7 ui: incident-bot /runbook-compile command and RunbookDraft renderer
- 5e6f7g8 tests: unit + integration + sanitize regression tests
- 6f7g8h9 docs: operator guide and console help text

Files/areas touched (high level):
- services/runbook-compiler/* (new service)
- services/telemetry-stitcher/* (new worker)
- services/ephemeral-debug-proxy/* (new sandbox proxy)
- web/incident-bot/commands/runbookCompile.ts (command wiring)
- web/console/components/RunbookDraft.vue (UI)
- infra/k8s/runbook-compiler-chart.yaml (helm chart)
- infra/policies/sanitizer-config.yaml (default sanitizer config)
- test/integration/runbook_compiler_smoke_test.go

Testing and CI:
- Unit tests for compilation heuristics and sanitizer rules (coverage: ~86%).
- Integration smoke: runbook compile end-to-end against a local stitcher and a mocked telemetry backend. One intermittent e2e flake (related to external trace service timeout) was retried and passed; we added a retry wrapper to the e2e harness.
- Load test: a small synthetic load harness that validates that the compiler can handle 40 RPS with default batching and falls back gracefully when quota exceeded.

Review conversation summary:
- Priya: Requested clearer failure semantics for the compiler when telemetry is partially available. Action: added a 'degraded' runbook mode and explicit indicators in the output.
- Jonas: Asked for tighter sanitizer controls and audit logs for sanitized artifacts. Action: added audit events and a sanitizer-config change log.
- Evan: Concerned about proxy resource exhaustion. Action: added a global supervisor with circuit-breaker and a maximum replay window config.
- Follow-ups left: add a small linear ticket to instrument cost metrics for runbook compilation (ENG-8622).

Backward compatibility and rollout plan:
- Feature is behind a feature flag (runbook.compiler.enabled = false by default in prod). We will enable in staging for two weeks and run the sampler to compare generated runbooks against existing manual playbooks.
- Canary rollout via the console: 10% of incidents for the paid tier, monitor TTR and false-positive step suggestions.

Operational notes for SRE and security:
- New k8s resources live in the runbook namespace and have poddisruptionbudgets set.
- Sanitizer config is stored in vault path secret/data/incident/sanitizer. Changes to that path trigger an alert to sec-ops.
- We’ve instrumented the following metrics: runbook.compile.duration, stitcher.correlation.count, proxy.replay.count, sanitizer.redactions.count. Dashboards and alerts added in observability-pack repo (PR coming).

Why this is different: Prior PRs focused on snapshot collectors or static playbook hints. This work is the first end-to-end compiler that (1) stitches telemetry slices, (2) emits a prioritized runbook with actionable safe rollbacks, and (3) provides a controlled sandbox to validate remediation steps without touching production state.

Notes for reviewers merging: squash merge; include the merge commit message template that references ENG-8421, ENG-8593. Merged to main after a green CI and approvals from two reviewers.

If anything is unclear or you want to run the sandbox locally, see docs/operators/runbook-compiler.md and the quickstart in services/runbook-compiler/README.md.
Adds a runbook compiler service and telemetry stitcher to produce compact, sanitized incident runbooks; introduces an ephemeral debug proxy for safe live-replay and rollback suggestion flows; no user-facing breaking changes.
