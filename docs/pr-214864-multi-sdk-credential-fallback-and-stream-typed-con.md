# multi‑SDK credential fallback and typed stream contract refactor

Motivation: several production deployments surfaced intermittent auth probe failures that caused streaming consumers to retry aggressively and lose backpressure semantics. This PR introduces a pluggable credential fallback chain, harmonizes a typed stream contract across Python/TypeScript/Go SDKs, and wires lightweight retry presets that prefer fast local fallbacks before hitting network refresh endpoints. The goals are: 1) make credential lookup resilient (local cache -> delegated proxy -> remote refresh) with a clear TTL/priority model; 2) propagate backpressure hints and a minimal typed envelope so SDKs can interoperate without behavioral surprises; 3) introduce small, opt-in retry presets tuned for streaming consumers to reduce noisy retries in client apps. Changes are split across SDK surfaces and the mono-repo coordination layer. See linked Linear items for rollout plan and metrics to watch. Checklist: [x] design doc linked in Linear ENG-2191; [x] unit tests for credential fallback and stream contract in each SDK; [x] integration test matrix against the proxy dev environment; [x] telemetry hooks for credential-fallback-count and stream-backpressure-events; [x] doc snippets for migration and example usage; [ ] follow-up: opt-in feature flag for legacy behavior for 1 release cycle. Backwards compatibility: default behavior is compatible but exposes a new opt-in policy field on client constructors. Rollout plan: merge to main, staged canary on Dedicated infra, 48h metrics review, then wide rollout.
initial: add credential fallback interface and simple chain implementation
ts: typed stream envelope and backpressure hint propagation
py: credential chain wiring and retry preset for long-lived iterators
go: stream contract adapter and prefetch-aware retry hook
tests: add cross-sdk contract conformance tests
docs: add migration notes and examples
fix: address race in cache refresh and add jitter to ttl refresh
Marco Rivera: please add a short paragraph in the docs about how the credential priority is resolved when multiple sources return a value.
Aisha Patel: added a new section in docs and a diagram showing priority order and TTL behavior.
Priya Shah: the TypeScript async iterator lost the backpressure hint in one branch; please ensure the hint is forwarded on pause/resume.
Aisha Patel: fixed the iterator branch and added a unit test that asserts hint propagation on pause/resume.
Daniel Cho: race detected in Go fallback cache leading to double refresh under high concurrency; add a singleflight-style guard.
Aisha Patel: implemented singleflight guard in fallback.go and added concurrency test; small performance hit is measured but acceptable for correctness.
CI: re-run cross-sdk contract matrix after fixes; all green.
Merged by squash-and-merge. Combined commits into a single change with a detailed commit message describing: credential fallback chain, typed stream envelope, retry presets, tests, and docs. Canary started after merge and observed a 2x reduction in streaming retry storms for the canary cohort.
Introduce pluggable credential fallback chain and typed stream contract across SDKs. New opt-in client policy for fallback behavior and streaming retry presets. Backward compatible by default. See docs/sdk-migration/ for migration steps and examples.
Enable cross-sdk contract matrix in CI for at least two CI cycles; coordinate canary with infra (Dedicated region eu-west-1) and monitor metrics: credential-fallback-count, stream-backpressure-events, consumer-retry-rate.
Added singleflight guard in Go and debounce on refresh in Python which introduce minor latency on first refresh but reduce overall refresh thundering; observed 3% median P50 increase on auth latencies in local test, but 30% lower retries for clients.
