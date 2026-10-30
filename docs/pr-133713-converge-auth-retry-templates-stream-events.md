# Cross-SDK consolidation: auth policy matrix, retry-templates and streaming adapter events

Summary: This PR brings three coordinated changes across the TypeScript, Go and Python SDKs: (1) introduce a small, opinionated retry-templates API to consolidate common retry/backoff profiles; (2) add a lightweight streaming adapter events surface to improve reconnect/heartbeat ergonomics for consumers; and (3) publish an AuthPolicyMatrix helper (Go) used by the private deploy installer and SDK session factories. Motivation: teams repeatedly re-implement slightly different retry/backoff tables and streaming reconnect hooks in each SDK; this results in subtle differences in behavior across languages and increases debugging overhead when users hit auth edge-cases or stream hiccups. This change aims to reduce cognitive load for integrators and make cross-language behavior predictable without forcing a single upstream runtime change.

What changed (high level):
- TypeScript: introduced src/retry/templates.ts (exposes templated RetryProfile objects and a builder), added streaming adapter events (stream/adapter.ts) with onReconnect/onPrefetch hooks, and bumped package.json to v2.1.0.
- Python: added redwood/retry/templates.py (same profiles + migration shim), updated async streaming consumer to accept event callbacks, and bumped to 3.2.0.
- Go: added auth/policy_matrix.go (AuthPolicyMatrix helper), exposed session factory constructor convenience overloads, and bumped to 1.9.1.
- Cross repo: added integration tests under ci/integration/sdks/stream-reconnect.test.*, updated docs at docs/sdk/migration-auth-retries.md and added a short migration example to examples/sdks/streaming-retries.

Checklist:
- [x] Design doc (ENG-8721) and informal sync with SDK leads
- [x] Unit tests for retry templates in all three SDKs
- [x] Integration tests for streaming reconnect across TS+Py
- [x] Docs & migration guide
- [x] Backwards-compatible API additions only (no breaking renames)

Commits (summary): feat(ts): retry-templates + streaming events; feat(py): retry templates + async event hooks; feat(go): auth policy matrix + session factory ergonomics; test(ci): add cross-sdk connect/reconnect harness; docs: migration notes and examples.

Files changed (top-level): sdk/typescript/src/retry/templates.ts; sdk/typescript/src/stream/adapter.ts; sdk/typescript/package.json; sdk/python/redwood/retry/templates.py; sdk/python/redwood/stream/events.py; sdk/python/setup.cfg; sdk/go/auth/policy_matrix.go; sdk/go/session/factory.go; docs/sdk/migration-auth-retries.md; examples/sdks/streaming-retries/ts-example.md; ci/integration/sdks/stream-reconnect.test.ts; ci/integration/sdks/stream-reconnect.test.py; README snippets and a couple of test fixtures.

Review conversation (high level):
- Priya: "Thanks — overall looks good. Can we make the TS retry templates serializable so customers can persist chosen profiles for debugging?"
- Maya: "Added to the template builder — added to_commit 5 and included a JSON schema for the templates."
- Liam: "Concern: Go AuthPolicyMatrix might encourage mixing responsibility between installer and SDK. Can you add a clear doc and keep the helper out of critical path?"
- Maya: "Matrix is a pure data helper and not used by default session creation; updated docs to call it an optional policy helper and moved runtime wiring behind a factory flag."
- Diego: "Integration tests flaked initially on CI due to timing noise in the reconnect test."
- Maya: "Reduced sleep windows, added a jitter seed in tests, and made the assertion resilient to NRT scheduling. CI re-run passed."

CI and stability: initial CI run failed on flaky integration test (timing assertion). Fixed, re-ran and all checks passed (unit, lint, cross-sdk integration). Merge notes: squashed and merged using 'squash' strategy to keep main history tidy. Post-merge action items: - Publish SDK packages (TS v2.1.0, Go v1.9.1, Python v3.2.0) after release approvals - Monitor streaming reconnects and retry-profile usage via telemetry (added a lightweight event to the SDK telemetry payload when a retry-template is used) - Follow-up ENG task to evaluate if any templates should be suggested by Redwood Optimize for specific workloads (ENG-8730). Overall risk: medium (cross-language surface area) but changes are additive and covered by tests and docs.
TypeScript SDK v2.1.0: add retry-templates and streaming event hooks
Go SDK v1.9.1: expose AuthPolicyMatrix and ergonomic session factory
Python SDK v3.2.0: unify retry defaults with templates and clarify backoff semantics
feat(ts): retry-templates + generator adapter
feat(py): retry-profiles + async event hooks
feat(go): auth policy matrix & session factory ergonomics
test(ci): cross-sdk reconnect harness
docs: migration guide and example updates
Priya Nair: can we make TS retry templates serializable for audit/debugging?
Maya Chen: added JSON schema and serialization helpers in templates.ts
Liam O'Connor: move AuthPolicyMatrix out of critical path and document optionality
Maya Chen: updated docs and gated wiring behind factory flag
Diego Alvarez: CI reconnect test flaked under noisy runners
Maya Chen: reduced sleep windows, added jitter seed and made assertions resilient
