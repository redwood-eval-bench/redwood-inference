# Session-anchored auth frames and origin-verified preflight for streaming tool calls

Motivation: streaming tool/function-calling paths were fragile under client reconnects and cross-origin environments. Scoped bearer tokens used to authorize tool calls are short-lived and bound to a connection; browsers that reconnect (SSE/WebSocket fallbacks) or intermediary proxies could inadvertently drop or replay Authorization headers leading to tool-call failures or accidental privilege elevation. This PR introduces a session-anchored auth "frame" concept plus a conservative, origin-verified preflight behavior to improve streaming stability and preserve OpenAI-compatible surface expectations.

What this change does: added a lightweight, signed session auth frame carried in a single header (X-Redwood-Auth-Frame). The frame contains minimal claims (session_id, scope, expiry, key_id) and is signed with the project's rotating signing key (JWS compact). Servers validate frames against the project's public keys and ensure the session_id matches the connection lifecycle. For older clients we continue to accept Authorization: Bearer tokens on /v1/* endpoints; however the server will preferentially validate X-Redwood-Auth-Frame for streaming endpoints to avoid header loss on intermediate reconnects.

CORS & preflight: added stricter handling for browser preflight when streaming endpoints are used. The server now returns Access-Control-Expose-Headers: X-Redwood-Auth-Frame and Access-Control-Allow-Origin is verified against the project's configured allowed_origins metadata. Preflight failures return 403 with structured JSON explaining the mismatch to help debugging.

Compatibility: to ease rollout this PR implements a compatibility shim: clients can send X-RW-Signed-Nonce (short single-use signed nonce) or the new auth frame. The shim is opt-in by header order: frame -> signed-nonce -> Authorization header. SDKs (TS/Python) are updated in companion commits to attach the auth frame automatically when a session manager is available.

Security model: frames are short-lived (default 2m) and bound to session_id. The server rejects frames that do not match the live session binding (on the connection handshake) to mitigate token replay across connections. Keys are rotated via the existing project key API; servers cache public keys with TTL and fall back to live lookup on verification failures.

Testing & rollout: added unit tests for parsing/verification, integration tests with SSE reconnect, and an e2e scenario exercising tool/function call acceptance after browser reconnect. Rollout plan: 5% canary (10 nodes) -> 25% -> 100% over 72 hours; monitor errors/api_auth.failure_rate and streaming.tool_call_drop_rate. Old header deprecation: Authorization-based inference will be deprecated for streaming after 30 days; the non-streaming REST surface continues to accept Authorization for 90 days.

Checklist: 
- [x] Add auth frame parsing and verification in runtime
- [x] Implement origin verification in preflight handler
- [x] Add compatibility shim for signed-nonce
- [x] Update TS/Python SDKs to attach frame when available
- [x] Add unit/integration/e2e tests
- [x] Add docs + migration notes

Related: ENG-4302 (session auth), PM-812 (streaming reliability), small overlap with previous auth header work but this is intentionally session-first and origin-aware.
Priya: can we make frame expiry configurable per-project? responding: added ProjectConfig.SessionFrameTTL with default 2m (see verifier.go)
Diego: ensure key rotation fallback doesn't cause latencies on hot paths. reply: verifier caches keys for 5m and does background refresh; fallback only on verification failure and logged (metrics: auth.key_lookup_fallback)
Tom: e2e was flaky locally, add retry for SSE reconnect in test harness. reply: adjusted harness and re-ran CI
Introduce session-anchored auth frames (X-Redwood-Auth-Frame) for more robust streaming and tool/function-calling. Add origin-verified preflight behavior and compatibility shim accepting X-RW-Signed-Nonce. Rollout is gradual; Authorization header remains accepted for REST endpoints during migration. See docs/auth/session-frames.md for migration steps.
This PR is intended to reduce tool-call failures observed in customer telemetry where browser reconnects lost Authorization headers. Metrics to watch post-merge: streaming.tool_call_drop_rate, api_auth.failure_rate, auth.key_lookup_fallback_count. SDK maintainers should publish new minor releases to pick up auto-attaching behavior.
