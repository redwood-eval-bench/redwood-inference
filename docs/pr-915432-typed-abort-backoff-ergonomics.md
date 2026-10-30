# introduce-typed-abort-and-named-backoff-presets-across-sdks

Motivation: teams using polyglot services reported inconsistent cancelation ergonomics and subtly different retry/backoff configuration across SDKs. This PR standardizes a small, typed AbortToken abstraction and a named backoff profile surface (fast, balanced, conservative) across Python, TypeScript, and Go. The change is ergonomic and additive; legacy config continues to work. Key changes: Python: add redwood.sdk.abort.AbortToken and redwood.sdk.backoff.Profiles; streams accept optional abort_token. TypeScript: export AbortToken type and BackoffProfiles in top-level package; streaming AsyncIterable constructors accept an abortToken. Go: introduce abortkit.AbortToken and backoff.Profiles struct; streaming client accepts ctx plus optional AbortToken. Shared artifacts: canonical JSON schema for backoff profiles and shared test vectors used in SDK unit tests. Ergonomics: cancelable streaming facades with concise examples, migration helpers to wrap legacy config into named profiles. Version bumps: python-sdk -> 3.4.0, typescript-sdk -> 2.2.0, go-sdk -> 1.7.3. Docs: cross-SDK quickstart demonstrating identical cancel semantics. The server contract is unchanged; AbortToken is advisory and non-serializable.
Adds AbortToken cancel hint and BackoffProfiles presets across Python/TypeScript/Go SDKs; minor version bumps. Backwards compatible by default.
Diego Ramos: Nice minimal surface. Two asks: ensure AbortToken serializes to nothing and add TS type-only import rule.
Maya Chen: Implemented non-serializable AbortToken in Python (__getstate__ raises) and added TS type-only export; added unit to assert non-serializable.
Avery Patel: Backoff presets look good. Request: expose profile_to_timeline debug utility and run perf-canary with 'balanced'.
Maya Chen: Added profile_to_timeline in Python/TS and Profiles.ToTimeline in Go; attached perf-canary timing vectors for 'balanced'.
Sofia Ivanov: Ensure abort hints are redacted from audit/event payloads.
Maya Chen: Updated audit serializer to drop AbortToken traces and added integration test for redaction.
CI feedback: initial run flagged TS lint and go vet unused import; fixed in follow-up commit.
Non-serializable AbortToken and audit redaction validated by unit and integration tests. Perf microbench shows <1% overhead when AbortToken unused.
Migration helpers provided in each SDK; see docs/sdk/cancelation-quickstart.md for examples that wrap legacy retry/backoff config into named profiles. No forced changes required.
SDK: add AbortToken cancel hint and BackoffProfiles presets (fast/balanced/conservative). Minor version bumps: Python 3.4.0, TypeScript 2.2.0, Go 1.7.3.
Added unit tests for abort non-serialization, profile timelines, and integration smoke for cancelation. Perf microbench confirms negligible overhead.
