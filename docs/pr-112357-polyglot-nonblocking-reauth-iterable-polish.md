# polyglot SDK: nonblocking reauth shim + safer async iterables and semver polish (py/ts/go)

Motivation: Multiple SDKs exposed slightly different streaming / iterator ergonomics and had several modal token refresh behaviors that caused blocking stalls during long-lived streams. This PR unifies the consumer-facing async iterator surface across Python/TypeScript/Go, introduces a nonblocking reauth shim that refreshes tokens opportunistically in the background and falls back to a short-lived reauth handshake only when necessary, tightens default retry/backoff settings for streaming reconnects, and applies minor semver bumps across SDKs.

The goals are: 1) avoid user-visible stalls during token rotation on long streams; 2) provide consistent iterator behavior and simpler migration; 3) ship small compatibility-safe version bumps.

Changes:
- Add background token refresher (nonblocking) to each SDK client; exposes a configuration hook (token_refresh_window_ms) and a guard that prevents concurrent blocking reauths.
- Rework async iterator/shim in Python to return an AsyncGenerator that implements a .cancel() handle and consistent error translation.
- Align the TypeScript AsyncIterable protocol to yield TypedEvent objects (same shape as Python and Go) and improve stream backpressure by batching small frames.
- Fix a go-stream sink leak that kept goroutines alive when the consumer dropped the iterator; add a pooled context cancel pattern.
- Normalize default retry/backoff to an exponential cap with jitter and shorter initial retry for 401->reauth flows.
- Bump versions: python -> 2.8.0, typescript -> 1.9.0, go -> 0.12.1.

Migration notes:
- Most users will not need to change call sites. If you relied on the old streaming error types, switch to the TypedEvent union; a compatibility shim is provided for one release.
- If you implemented custom token rotation by intercepting low-level sockets, opt into token_refresh_window_ms or disable background refresh.

Checklist:
- [x] Implement background refresher in py/ts/go
- [x] Add unit + integration tests for long-lived streams and auth rotation
- [x] Add cross-SDK typed event shape spec
- [x] Update SDK CHANGELOGs and bump versions
- [x] Validate end-to-end with hosted API Canary
- [x] Docs snippet for migrating stream error handling

Commits:
- "feat(sdk): add background nonblocking token refresher and config hook"
- "refactor(py): async generator cancel handle + deterministic teardown"
- "feat(ts): align AsyncIterable event shape + small batching optimization"
- "fix(go): close sink goroutine on consumer drop (pool cancel pattern)"
- "chore: unify retry/backoff defaults for streaming 401 flows"
- "test: add long-lived stream token rotation integration test"
- "chore: bump sdk versions and CHANGELOGs"
Connor Liu: Left a few inline nits but overall this looks solid. Can we tighten the default token_refresh_window_ms to 30s instead of 60s? Reason: hosts with short live tokens saw stalls in canary.
Maya Patel: Updated to 30s and added a comment in the config explaining tradeoffs. Also added a test that simulates 5s token TTL in CI integration harness.
Janelle Rivera: Approve after the docs snippet is added and CHANGELOG entries are present. Please call out the compatibility shim release in the notes.
Arjun Mehta: Noted one potential race in Go pool cancel; added a suggested patch to ensure idempotent Close. Patch applied in commit 4th.
CI Bot: Integration test failed on first run (timeout in long-lived stream); rerunning...
CI Bot: All checks passed after rerun.
