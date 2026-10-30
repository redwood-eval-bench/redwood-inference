# introduce-typed-stream-bridge-and-compact-retry-profiles-for-tri-sdk-parity

Motivation: SDK consumers reported subtle mismatches in streaming ergonomics and retry semantics across Python, TypeScript and Go. This caused odd edge-case reauth and backoff behavior in long-lived streams. This PR introduces a lightweight typed-stream bridge contract and a compact retry profile bundle that each SDK can adopt to achieve parity without duplicating orchestration logic.

What this PR does (high level):
- Adds a typed-stream bridge specification in /specs/sdk/typed_stream_bridge.md and a compact JSON retry-profile catalog at /specs/sdk/compact_retry_profiles.json (consumers: py/ts/go SDKs).
- Implements bridge adapters in the runtime API surface (redwood/sdk-contracts) for aligning cursor semantics, heartbeat hints, and backpressure cues.
- Updates SDK-facing examples and test matrices for Python, TypeScript and Go to demonstrate behavior with the compact retry profiles (adaptive-window, conservative, optimistic).
- Harmonizes default retry/backoff parameters and exposes a compatibility flag for legacy behavior.
- Bumps SDK compatibility matrix entries in docs and emits changelog fragments for all three SDKs.

Why this approach: Centralizing a small typed contract and reusable compact profiles lets us iterate on backoff defaults and stream cursors without cross-repo churn. SDK teams can adopt the spec by importing the JSON/profile and wiring the bridge adapter; no protocol wire change required.

Compatibility and rollout: The default remains backward-compatible; adopters can opt-in to the new bridge via client config: {"stream_bridge": "v1-compact"}. A feature flag on the server side will gate the richer cursor metadata until clients opt in.

Testing: Added unit tests in redwood/sdk-contracts, integration tests that spin up the local stream harness and run through reauth scenarios, and cross-SDK smoke tests validating that TokenRefresh events and Pause/Resume semantics are consistent across languages.

Notes for reviewers: focus on the spec doc for contract clarity, the compact retry profile values (especially adaptive-window math), and the bridge adapter API shape which will be imported by SDKs.

Checklist: Added tests; updated docs; added changelog fragments; validated CI on Linux/macOS; coordinated with SDK owners for follow-up PRs that will adopt the spec.
Jonah Reyes: Can we tighten the adaptive-window growth factor? In heavy contention it looks like the optimistic profile can double too aggressively.
Marina: Adjusted from 2.0 -> 1.5 and added a unit test to simulate congestion; see commit 'tune-adaptive-window'
Priya Shah: The TS adapter shape returns a nullable cursor in a few paths which complicates downstream typing. Can we return an explicit union type instead?
Marina: Converted nullable returns to a discriminated union (type StreamCursor = {kind: 'ok', cursor: string} | {kind: 'pending', nextHintMs: number}). Updated examples and test assertions.
Luca Bianchi: Go code uses time.Sleep in test harness which inflates CI time. Any chance to replace with a virtual clock?
Marina: Replaced the slow sleeps with the harness' mock clock; CI run dropped ~90s in integration stage.
