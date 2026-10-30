# streaming-backpressure-aware-client-cancel-idempotency-openai-compat

Motivation: In a few customer incident postmortems we observed two related issues during high-concurrency streaming: (1) client rapid-cancel storms could leave tool/function-call side effects duplicated or orphaned, and (2) strong downstream backpressure from certain SDK consumers caused partial JSON/structured outputs to be emitted and then retried, breaking downstream parsers. This PR implements a combined runtime + OpenAI-compat shim approach: per-stream backpressure windows to slow flushes under sustained pressure, an idempotent cancel token for client-initiated cancels (so retries or duplicate cancel events are a no-op), and a conservative structured-output integrity check that forces a partial-flush snapshot to be fenced until the next continuation is confirmed. High-level changes:
- stream_manager: add a backpressure windowing scheduler that applies exponential dampening on flush frequency per-client when writebuf latency exceeds threshold.
- cancel token: introduce a short-lived idempotency token returned in-stream when client requests cancel; server honors token once and records tombstone for 30s to dedupe duplicate cancels.
- openai_compat shim: map incoming "stop" and "abort" semantics from OpenAI-style clients to the idempotent cancel token flow and emit a stable "canceled" event to consumers.
- toolcall ordering: attach per-toolcall dedup index and pre-commit checkpoint so that if a cancel races with a tool invocation we either roll back the pending invocation or mark it as completed exactly-once.
- structured output: when a flush would break JSON/object boundaries we now snapshot the in-progress structured envelope and mark it as "incomplete" in the event stream; a follow-up continuation with the same sequence id will be required to complete the object. This reduces parsers accidentally accepting malformed objects.
Checklist:
- [x] Design doc linked (linear ENG-91203)
- [x] Unit tests for cancel idempotency and tombstone expiration
- [x] Integration test with JS SDK showing client rapid-cancel + retry does not duplicate toolcalls
- [x] Load test demonstrating reduced malformed structured output under backpressure
- [x] Docs update for OpenAI compatibility mapping
See also: docs/openai/compat-cancel-behavior.md and sdk/js/README.md (examples).
Commits:
- 9f3b2a4: add backpressure window scheduler and metrics hooks
- 2d8e1b9: implement cancel-idempotency token, tombstone store, and API mapping
- 6c7fa11: structured-output snapshot fencing and toolcall dedup index
- a1b4e2c: tests: unit and integration coverage for cancel and backpressure
- b6d8c3f: docs: openai compat and changelog entries
Testing & CI:
- Local unit test suite added: streaming/backpressure.test.ts, streaming/cancel-idempotency.test.rs
- Integration: sdk/js example that spawns 200 concurrent streams and issues randomized cancel/retry patterns.
- CI: all checks passed on the final run (lint, unit, integration, e2e load smoke).
Files changed at a glance:
- src/runtime/stream_manager.rs: add backpressure scheduler, flush dampening, metrics (adds ~230 lines)
- src/runtime/cancel_store.rs: new tombstone and token validation (adds ~80 lines)
- src/compat/openai_shim.rs: map OpenAI stop/abort -> idempotent cancel, adapter tests
- sdk/js/src/stream.ts: expose client-cancel token handling and retry guidance
- docs/openai/compat-cancel-behavior.md: compatibility matrix and examples
- tests/integration/streaming/cancel_race.test.ts: end-to-end reproduction harness
Review conversation (summary):
- Diego Ramirez: "Overall approach looks reasonable. Two main concerns: tombstone memory growth and the interaction with persistent Dedicated routing. Please add eviction and metrics for tombstone set size."
- Author (Maya): "Added a TTL-based LRU eviction and prometheus-style metric stream.tombstone_count; included in 9f3b2a4/2d8e1b9. Noted the Dedicated routing edge case and gated the tombstone store behind per-tenant config in the shim."
- Priya Kapoor: "Toolcall dedup is good, but I'm worried about losing responsiveness for low-latency SDKs when we backpressure too aggressively."
- Author (Maya): "Backpressure window is conservative out-of-box (100ms min dampening) and can be tuned per-client; added integration test to prove latency tail stays within SLOs under normal load."
- Liam O'Connor: "Tests look solid; can we add a quick migration note for customers using custom OpenAI-compat logic that depend on previous 'raw stop' semantics?"
- Author (Maya): "Added docs/openai/compat-cancel-behavior.md with migration guidance and sample shim to reproduce old behavior if explicitly enabled."
Add per-stream backpressure window and idempotent client-cancel token; map OpenAI-style cancel signals to idempotent abort semantics; preserve structured output integrity across partial flushes.
Diego Ramirez: Overall approach looks reasonable. Two main concerns: tombstone memory growth and the interaction with persistent Dedicated routing. Please add eviction and metrics for tombstone set size.
Maya Chen: Added a TTL-based LRU eviction and prometheus-style metric stream.tombstone_count; included in 9f3b2a4/2d8e1b9. Gated tombstone store behind per-tenant config for Dedicated.
Priya Kapoor: Toolcall dedup is good, but worried about losing responsiveness for low-latency SDKs when we backpressure too aggressively.
Maya Chen: Backpressure window defaults to conservative values (100ms min dampening). Added integration tests proving tail latency remains within SLO for regular traffic.
Liam O'Connor: Tests look solid; add migration note for customers depending on previous raw stop semantics.
Maya Chen: Added docs/openai/compat-cancel-behavior.md with migration guidance and explicit flag for legacy stop semantics.
