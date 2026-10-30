# client-cancel-debounce-and-server-grace-lease-to-reduce-cancel-avalanche-openai-compat

Motivation: We observed production incidents where brief client churn (mobile signal flaps, browser tab swaps) produced large numbers of client-cancel signals that cascaded into unnecessary work abandonment, toolcall retries, and thundering fallbacks. This PR introduces a client-side cancel debounce, a server-side micro-grace lease extension, and improved toolcall idempotency propagation for OpenAI-compatible streaming endpoints to reduce cancel avalanches and make short disconnects survivable.

What this change does (high-level):
- Client: adds a 250ms soft-cancel debounce before sending an explicit cancel over streaming and websocket transports. The client will still send a hard cancel if the connection is explicitly closed by user action (explicit stop button) or a second cancel within the debounce window occurs.
- Server runtime: when a transient cancel or TCP close is observed, the stream controller applies a micro-grace lease (default 500ms, configurable per route) that delays final execution abort. The runtime will attempt to quiesce any in-flight toolcall (if idempotency token is present) and perform a lightweight checkpoint (partial KV-cache snapshot) to allow resume without restarting the model run.
- OpenAI-compat: we add support for the X-OpenAI-Compat-Cancel-Mode response header and accept an idempotency token header X-OpenAI-Compat-Idempotency to tie toolcalls to retry-safe semantics. This keeps compatibility with existing OpenAI-like clients while giving explicit control for idempotent tool invocation.
- Observability: new metrics (stream.cancel_debounced, stream.grace_lease_issued, toolcall.idempotency_hit) and tracing spans to correlate cancels to final aborts and tool retries.

Design notes and alternatives considered:
- We considered only server-side smoothing, but that meant clients could still generate huge numbers of cancel messages that would hit the control plane. The combined debounce + server grace offers better end-to-end stability with minimal protocol surface change.
- Debounce values chosen conservatively based on production histograms (P50 ~ 120ms transient disconnects, many reconnects under 300ms). We picked 250ms/500ms for default; both are route-configurable.

Checklist: 
- [x] client debounce implementation (TS/py SDKs) and tests
- [x] runtime guard (C++/Rust paths) with param-driven grace lease
- [x] header-level OpenAI-compat glue for idempotency and cancel-mode
- [x] unit/integration tests and flaky-test markers
- [x] metrics and docs update

Testing performed:
- Unit tests for debounce timers and cancellation state machines (both SDKs)
- Integration harness that simulates bursty disconnects and verifies reduction in active aborts and tool retries (streaming harness + mock tool server)
- Canary on staging with 24h traffic replay; observed a 48% drop in aborted toolcalls under synthetic churn.

Backward compatibility and migration:
- Default behavior is backwards compatible: clients that do not send idempotency headers or do not implement SDK debounce see no functional change. Servers continue to accept existing cancel semantics but add the grace lease server-side.
- Operators can disable grace lease per route via config if they prefer immediate abort semantics for some workloads.

Open questions / follow-ups: 
- Should debounce windows be adaptive to RTT for mobile clients? (tbd, follow-up ENG-2472)
- We plan to add a debug trace flag to force immediate aborts for debugging purposes (opt-in).

Summary: This PR reduces noisy abort traffic, improves toolcall retry economics via idempotency propagation, and makes short client disconnects survivable without changing the openai-compatible streaming contract in ways that break clients.
Diego Alvarez: Can we make the grace lease configurable per-model and per-route? I think larger models may need longer windows. - Addressed by adding route config and docs.
Priya Nambiar: We need to ensure the checkpoint path doesn't add meaningful latency to the fast-path. How heavy is the snapshot? - Author: we do a lightweight KV cache pointer copy and async persist; no blocking on token generation.
Eliot Turner: Add an e2e test that asserts idempotent toolcall is not retried twice on reconnect. - Added integration test and mock tool server assertion.
Diego Alvarez: please add a migration note for operators wanting immediate aborts. - Added: operator toggle documented in docs/streaming/cancel_grace_and_idempotency.md
Introduce client-side soft-cancel debounce and server micro-grace lease to reduce cancel avalanches; add OpenAI-compat idempotency header and cancel-mode response header; new metrics for cancel and toolcall idempotency. Defaults are backward compatible and configurable per-route.
Operators: a new per-route config `stream.cancel_grace_ms` defaults to 500ms. SDKs ship debounce (250ms) behind minor-version opt-in; rolling upgrade recommended for low-latency apps.
