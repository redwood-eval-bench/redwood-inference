# harmonize-cred-fallback-and-typed-stream-handles-bump-v1.6.0

Motivation: customer teams saw divergent retry behavior and credential fallback semantics between Go and the other SDKs which created intermittent auth failures under heavy streaming load. This PR harmonizes credential fallback ordering, surfaces a lightweight typed stream handle for cancel/drain ergonomics, and updates the Go SDK to v1.6.0 to match Python/TS parity work.

What this PR does (single-string body): Adds a CredOrchestrator that composes primary token source + delegated fallback providers with deterministic priority and backoff hints; introduces StreamHandle (typed, context-aware) that exposes Cancel/Drain/Closed semantics and emits structured stream lifecycle events; surfaces declarative RetryProfile presets (hedged, latency-sensitive, throughput) and plumbs them into the client transport; fixes several race conditions in session renewal (session.go) discovered in integration tests; updates examples and changelog; bumps module versions and tags v1.6.0. Checklist: [x] unit + integration tests updated, [x] cross-sdk spec docs referenced, [x] minimal migration notes in CHANGELOG, [x] version bump and release notes prepared. Migration notes: existing code using RawStream() can swap to StreamHandle via client.Stream(ctx, opts) which returns an idiomatic interface with Cancel() and WaitClosed(ctx). Backwards compatibility shim provided for one release.
3 commits:
- introduce cred orchestrator, retry presets, and StreamHandle (amit/feature/cred-orch-streamhandle)
- fix session renewal races, add integration test harness (amit/fix/session-renewal-race)
- docs/examples + changelog + version bump to v1.6.0 (amit/release/v1.6.0)
Small illustrative snippets: added StreamHandle interface with Cancel/WaitClosed; cred_orchestrator implements Fallback() and Prefetch(); retry_profiles expose templates (Hedged, LatencySensitive, CostAware). No large diffs; most logic ~100-150 LOC additions per core file.
Lina Chen: Nice consolidation. Two asks: (1) make Prefetch window configurable via client opts (2) ensure fallback ordering is documented with a sequence diagram.

Author (Amit Bhatt): Addressed both: added ClientOptions.CredPrefetchMs and included an ASCII sequence diagram in auth/README.md. PTAL.

Owen Martinez: Found a subtle goroutine leak when stream.Close() races with credential refresh in session.go. Suggested using a dedicated cancel group and adding a unit test.

Author: Reworked session shutdown to use errgroup with explicit cancel propagation and added a unit that reproduces the previous leak; test is deterministic under the harness.

Priya Singh: Integration tests flaky on CI (timing). Recommend making RetryProfile default timeouts conditionally larger in CI.

Author: Added CI-specific override in .github/workflows/ci.yml for the integration job to use profile.testTimeout multiplier; reran CI.

Lina Chen: Approve after CI green; thanks for the clear migration guide.

Go SDK v1.6.0: Harmonized credential fallback ordering with Python/TS, added StreamHandle (Cancel/WaitClosed) for safer consumer ergonomics, new RetryProfile presets, session renewal race fixes, migration shim for RawStream() compatibility.
