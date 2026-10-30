# Introduce debounced auth adapter, backpressure-aware stream consumer, and compatibility version bump

Motivation: production customers running bursty streaming workloads observed repeated token refresh handoffs and transient auth failures when many lightweight streams re-auth concurrently. Separately, Go SDK stream consumers did not expose a backpressure signal compatible with recent TS/Python adapters which led to suboptimal memory growth and rebuffering during spikes. This PR introduces a lightweight DebouncedAuthAdapter and a BackpressureStreamConsumer that together smooth credential exchanges and provide a pluggable signal for flow control.

What this change does (single continuous description): Adds DebouncedAuthAdapter (configurable debounce window, jitter, and max-concurrency) that centralizes token refresh requests per-client to avoid stampeding herd effects. Introduces BackpressureStreamConsumer which exposes a small interface for producers to observe consumer lag and adapt batching/emit rates; provides two ready-to-use strategies (linear decay and token-bucket). Adds a thin compatibility shim to align the Go adapter surface with the new TS/Python backpressure hints (parity interface only, opt-in). Updates docs and examples with migration notes and an example that wires DebouncedAuthAdapter into the streaming client. Includes unit tests for debounce timing, concurrency limits, and integration tests that simulate 500 concurrent short-lived streams with synthetic auth churn.

Checklist: [x] unit tests added; [x] integration load test added; [x] docs/example updated; [x] CI passing after two reruns; [x] reviewers approved.

Design notes: The DebouncedAuthAdapter maintains a single inflight refresh per credential key and queues duplicate refresh requests within the debounce window. This reduces downstream reauth calls to the network and keeps token rotation windows tight. BackpressureStreamConsumer exposes ConsumerLag() and Ack(n) primitives; it does not mandate a particular scheduling algorithm so server-side runtime can implement adaptive batching. We intentionally kept the public surface minimal to avoid breaking existing users — adapters are opt-in and default behavior is unchanged.

Compatibility and rollout: Feature gated behind a constructor option in the streaming client. Patch bumps SDK minor version to v1.8.0 and documents the opt-in nature. Cross-SDK parity: collaborated with TS and Python SDK teams (linked Linear tickets) to align signal names and semantics.

Related: ENG-4021 (debounce auth), ENG-4098 (stream backpressure), SUPPORT SUP-3312 (customer incident reproduction), design notes link: https://linear.app/redwood/issue/ENG-4021/debounce-auth-adapter
Go SDK v1.8.0: add DebouncedAuthAdapter to reduce auth thrash, introduce BackpressureStreamConsumer to provide adaptive buffer/ack behavior for high-throughput streaming, and bump compatibility helpers for cross-SDK parity. No API surface breaking changes; opt-in features only.
Marco Li: Overall looks good. Two concerns: make sure the debounce window default is conservative (10s) and add a metric for dropped refresh requests.
Aisha Patel (author): Increased default debounce to 10s, added metrics 'auth.debounce.requests' and 'auth.debounce.dropped' and a README note about tuning.
Leah Thompson: Can BackpressureStreamConsumer's ConsumerLag be computed without locks (atomic window)? Consider using a ring buffer snapshot to avoid pause under load.
Aisha Patel (author): Switched lag calculation to an atomic ring snapshot for low-latency reads. Added microbenchmark showing <2us median read under 1000rps.
Jonas Müller: Integration tests are heavy but useful. One nit: call out that the feature is opt-in in the top-level README and in the changelog.
Aisha Patel (author): Added changelog entry and top-level README note. Marking as ready for squash-merge.
