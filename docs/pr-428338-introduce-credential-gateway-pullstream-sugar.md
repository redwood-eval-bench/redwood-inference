# Introduce credential-gateway + pullstream-sugar: compact session routing & ergonomic async generators

Summary: This PR introduces a lightweight 'credential-gateway' component plus a set of 'pullstream-sugar' adapters that provide ergonomic async-generator/pullstream facades across the Python, TypeScript, and Go SDKs. The goal is to centralize credential selection & prefetch logic (local + delegated chains) and to expose a small, consistent async iteration surface for streaming responses that reduces boilerplate in user code.

Motivation and context: Multiple teams reported friction wiring credential selection into streaming callers: duplicated prefetch logic, inconsistent failure semantics during mid-stream reauth, and divergent ergonomics across SDKs. Instead of further ad-hoc fixes in each SDK we built a thin orchestrator (credential-gateway) that: 1) mediates credential lookup and fallback policies, 2) exposes a compact lease/claim API for streams, and 3) exposes hooks used by language adapters. The pullstream-sugar adapters implement a small ergonomic wrapper (async generator in Python/TS, channel-like iterator in Go) that: - automatically primes credentials for a session, - surfaces a single typed StreamHandle with .close/.pause hints, and - reconciles retry/backoff signals from the runtime into a stable consumer contract.

What changed (high-level):
- Introduced internal/cred/gateway.go and internal/cred/gateway_pyshim.py: core routing and lease semantics.
- Added sdk/python/redwood/pullstream_sugar.py: ergonomic async generator + examples.
- Added sdk/ts/src/pullstreamSugar.ts: compact async iterable wrapper and TypeScript declarations.
- Added sdk/go/pullstream/sugar.go: channel-backed stream adapter and small helper API.
- Bumped SDK versions and package metadata: python 1.13.0, typescript 2.8.0, go v1.5.2.
- Tests: new unit tests for gateway routing matrix + integration harness exercising mid-stream credential rotation and a mock runtime.
- Docs: SDK examples updated with recommended streaming patterns and migration notes. Feature gated behind EXP_CRED_GATEWAY flag in env for gradual rollout.

Commits (summary):
- 1f4a3d3: introduce credential gateway core routing + lease semantics
- 9b2e7ab: add python pullstream_sugar adapter + examples
- 55d9c44: add typescript pullstream sugar + types
- 2a7f1b0: add go pullstream sugar adapter
- ad3c12f: tests for routing, prefetch, and mid-stream reauth behavior
- ff9e001: docs and migration guide
- cfe1234: bump SDK versions and changelogs

Files touched (high level):
- internal/cred/gateway.go
- internal/cred/gateway_pyshim.py
- sdk/python/redwood/pullstream_sugar.py
- sdk/python/tests/test_pullstream_sugar.py
- sdk/typescript/src/pullstreamSugar.ts
- sdk/typescript/test/pullstream_sugar.test.ts
- sdk/go/pullstream/sugar.go
- sdk/go/pullstream/sugar_test.go
- docs/sdk/streaming/pullstream-sugar.md
- scripts/feature_flag/EXP_CRED_GATEWAY.md
- packaging/* (version bumps)

Design notes & compatibility: The gateway is intentionally minimal: it does not persist long-lived tokens and it does not change customer key management — it delegates to configured KMS/hook providers. All public SDK surfaces are opt-in: the pullstream-sugar modules are additive (exported helpers), and existing streaming entry points continue to work unchanged. The common primitive is StreamLease (internal) which the adapters translate to idiomatic language constructs.

Testing & CI: Unit tests cover routing matrices and failure scenarios (prefetch fail, mid-stream refresh fail, fallback to next credential). Integration harness (flaky on first run) was stabilized by adding a deterministic mock kernel; CI now shows all checks green: unit, lint, typecheck, integration, e2e. Safety checks: lint/formatting applied across repos, go vet and gometalinter run, mypy for python. e2e was retried twice due to an unrelated infra flake; final run passed.

Review thread (selected excerpts):
- Mateo Alvarez (reviewer): "High-level: I like the separation. Two asks: (1) call out failure semantics when gateway cannot obtain any credential mid-stream, (2) ensure we don't spin new goroutines per request without bounding."
- Rina (author): "(1) added explicit docs section and a test asserting 'stream closes with credential_error' in that case. (2) gateway uses a bounded worker pool (gateway.go:line ~210) — I added a comment and a simple semaphore impl in the package."

- Lena Zhou (reviewer): "TS API looks ergonomic, but type coverage is missing for the consumer-facing .pause() hints. Also: please add a short migration snippet that shows turning existing generator usage into pullstream-sugar."
- Rina: "Added types and migration snippet in docs/sdk/streaming/pullstream-sugar.md, and added tests for pause/hint behavior."

- Omar Farouk (reviewer): "Can we avoid a breaking change for the Go SDK? The new sugar package should be optional and not shadow existing Stream() call signatures. Also, please run preflight against our internal 'edge' runtime image."
- Rina: "Confirmed: sugar.go is an additional package under sdk/go/pullstream; no signatures changed. Preflight run attached; small variant in edge runtime surfaced a timeout which I adjusted via a small backoff tweak (gateway.go)."

Requested changes resolved: semaphore pool comment, explicit error semantics test, types for pause hints, migration snippet, preflight backoff tweak. All resolved in follow-up commits.

Performance notes: Gateway adds a ~200-500 microsecond median overhead for the initial stream establishment (measured in the integration harness) but reduces cumulative token cost in heavy workloads via credential prefetch and reduced mid-stream churning. In local benchmarks with 512 concurrent streams the gateway reduced reauth storms by ~72% and improved tail latency for tokenized requests in the host runtime.

Backwards compatibility & rollout: Feature is behind EXP_CRED_GATEWAY. Gradual rollout plan: enable for internal customers first, then opt-in beta, then default in next minor release after telemetry confirms expected reduction in reauth failures. SDKs bumped but adapters are opt-in.

Follow-ups (future work):
- Add metrics for credential lease hold times and gateway rejection reasons (ENG-3501).
- Add server-side hinting API to pre-warm specific runtime shards (performance experiment).
- Consider a lightweight persisted cache for long-lived token refresh optimizations.

Merge: Squash merged to main on 2026-02-25 after approvals from reviewers and green CI. Change log and release notes updated across SDK packages.
Added credential-gateway (centralized routing for credential selection/prefetch), introduced pullstream-sugar adapters (ergonomic async generator wrappers) for Python/TS/Go SDKs. Bumped SDK versions: python->1.13.0, typescript->2.8.0, go->v1.5.2. No breaking API changes for existing consumers; opt-in feature behind feature flag.
