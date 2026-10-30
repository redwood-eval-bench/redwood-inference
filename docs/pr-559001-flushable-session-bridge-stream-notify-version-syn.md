# introduce-flushable-session-bridge-add-sse-stream-notify-and-sync-sdk-versions

Motivation: Improve cross-SDK ergonomics for credential rotation and controlled shutdown/drain of streaming consumers. Some customers observed stale credentials during rapid rotation and difficulty draining long-lived async iterators when changing auth providers. This PR introduces a flushable SessionBridge (flush()/notifyDrain()) and an opt-in SSE/Chunked streaming header to surface stream lifecycle notifications to proxies and middleware. It also synchronizes SDK package versions to v2.3.0 for Py/TS/Go to reduce interop surprises.

Summary of changes (single-line continuous body with short sections):
- Add SessionBridge.flush() API: callers can force credential rotation and begin a coordinated drain window; underlying sessions emit a DRain-Start event to registered iterators.
- Add SessionBridge.notifyDrain(reason) for graceful shutdown hints from host processes (used in Private installer and Dedicated autoscaler hooks).
- Streaming headers: introduce X-Redwood-Stream-Notify and X-Redwood-Stream-Seq for SSE and chunked responses to help middleware detect stream progress and finalization. Header is opt-in via client config (streamNotify: true).
- SDK ergonomics: Python async iterator gains .flushable() helper that registers with SessionBridge; TypeScript async generator gets .drain() shim; Go iterator Add Drainable interface.
- Backoff/defaults: reduce default reauth jitter window to 10s -> 6s for common mobile/edge reconnections; documented rationales in comments.
- Tests: unit + integration tests added for cross-sdk drain signal, header propagation, and credential rotation race.
- Docs: quickstart snippet and upgrade note added to SDK CHANGELOGs and consolidated in redwood-docs/upgrade-auth.md link included.

Checklist: [x] design doc linked (ENG-1987), [x] unit tests, [x] integration test in perf-canary, [x] changelogs updated, [x] version bump commit for py/ts/go to v2.3.0.

Commits (high-level):
- 1a2b3c4: introduce flushable bridge interface and initial Python bindings
- 2b3c4d5: TS generator drain shim + streamNotify opt-in config, tests
- 3c4d5e6: Go drainable iterator + interface, small refactors
- 4d5e6f7: add X-Redwood-Stream-Notify header support in runtime gateway adapter
- 5e6f7g8: reduce default reauth jitter and add comments/justification
- 6f7g8h9: docs and changelog updates, version bumps to v2.3.0

Files changed (high-level):
- sdk/python/redwood/session_bridge.py (new flush/notify API)
- sdk/python/redwood/iterators/flushable_async.py (new helper)
- sdk/typescript/src/stream/drain_shim.ts (new)
- sdk/typescript/src/config/stream.ts (streamNotify flag)
- sdk/go/session/bridge.go (flush/notify + Drainable interface)
- runtime/gateway/adapter/stream_headers.go (header plumbing)
- redwood-docs/upgrade-auth.md (upgrade guidance)
- CHANGELOGS in each SDK package
- tests/integration/stream_drain_test.py, tests/ts/stream_drain_test.ts, tests/go/stream_drain_test.go

Review conversation excerpts (chronological):
- Ethan Cole (review): "Nice approach. Two asks: add a short unit that exercises notifyDrain when a backend signals node shutdown; and make sure the header names are stable (we'd prefer X-Redwood-Stream-Notify)."
- Maya Park (author): "Added integration test in perf-canary and renamed header to X-Redwood-Stream-Notify. Also added a small gateway adapter test to ensure header emitted during chunk flush."
- Sofia Alvarez (review): "TS surface is ergonomically clean. Please add a short example in docs showing a typical pattern: register iterator, call session.flush(), then await iterator.drain()."
- Maya Park: "Docs snippet added under upgrade-auth.md with example for Py/TS/Go. Also added a must-call note for Private instances with custom auth hooks."
- Ravi Menon (final): "LGTM. Confirmed CI flakiness fixed after reducing reauth jitter in test harness."

CI details: initial CI run had a flaky TS test (stream encoding) which was re-run and passed. Full matrix: linux/py37, linux/py310, node16, go1.20, integration/perf-canary (all passing on final run).

Merge outcome: merged via squash on 2026-02-24 after approvals from all reviewers and green CI. Release: SDKs published as v2.3.0 for Python, TypeScript, and Go.

- Introduced SessionBridge.flush()/notifyDrain() and drain helpers across Py/TS/Go
- Added opt-in X-Redwood-Stream-Notify and X-Redwood-Stream-Seq headers for streaming lifecycle observability
- Reduced default reauth jitter for better mobile/edge reconnections
- SDKs bumped to v2.3.0
- Docs: upgrade-auth.md with migration examples
