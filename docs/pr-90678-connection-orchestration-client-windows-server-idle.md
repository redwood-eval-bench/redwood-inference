# Connection orchestration: client grace windows, server idle timeouts, and backpressure for streaming API (OpenAI-compat)

Motivation:
Client disconnects and slow clients were contributing to two classes of production issues: 1) sessions left open on the server (KV-cache and state) that consumed GPU memory until evicted, and 2) sudden resource pressure when many slowed clients resumed and pushed large request windows, causing transient tail-latency spikes. This PR introduces a coordinated connection orchestration layer for our OpenAI compatibility streaming surface to address both problems.

What this change does (single-pass summary):
- Adds a configurable client grace window on cancel/disconnect events (default 45s) where the server keeps ephemeral state but disables expensive prefetching and applies rate caps.
- Implements a server-side idle timeout with an active probe/ping before final teardown to avoid noisy double-teardowns and give clients a chance to reattach.
- Introduces a lightweight backpressure token bucket per-session to slow incoming token production from resumed clients and protect KV-cache growth.
- Propagates cancel semantics to in-flight function/tool calls while preserving idempotency metadata so downstream tool providers can safely retry or noop.

Design notes and rationale:
- We prefer a grace window vs immediate teardown to support intermittent mobile networks and short client restarts (common in SDK telemetry). The 45s default is based on routing team traces showing a median reconnect < 30s for mobile clients with occasional spikes.
- To prevent resource leakage, during the grace window we mark sessions as "drained" for background prefetch and large optimizer tasks; serving runtime keeps only minimal KV-cache checkpoint pages in pinned memory.
- Server idle timeout (configurable per deployment tier) uses a two-step sequence: (1) on reaching idle threshold, send a probe frame to the client and wait probe_timeout (default 10s); (2) if no ack, proceed to final teardown. This reduces race conditions when clients are closing and servers concurrently decide to evict.
- Backpressure is implemented as a token-bucket applied to the incoming token stream for resumed sessions. Bucket refill rate is derived from the session's SLO class (hosted/dedicated/private) and current GPU sched pressure. This prevents pathological bursts from causing preemption.

API and compatibility considerations:
- All behavior is gated behind the OpenAI-compat streaming layer flag; host customers keep the same wire protocol. SDKs will surface a new connection_lifecycle option to tweak grace_window_ms and enable_probe.
- Function/tool call cancellation: we now attach a per-call idempotency_token and a cancellation intent header so external tool adapters can decide to continue or abort. This is opt-in for adapters lacking idempotency guarantees.

Testing and verification:
- Unit tests: connection state transitions, token-bucket refill logic, and probe flow were covered.
- Integration: stress tests against the bench-lab with 10k simulated mobile clients with randomized disconnects, verifying that retained memory per-session is within expected bounds and tail-latency stays under SLO when resume storms happen.
- Canary: rolled to perf-canary (10% of traffic) for 48 hours; observed 38% reduction in leaked-session memory and 22% reduction in 99.9th token latency during reconnect floods.

Rollback plan:
- Feature is behind runtime flag and config knobs. If we see regressions, ops can flip the feature gate and the server will revert to prior immediate-evict behavior. We also added metrics (conn.orch.grace_evicted, conn.orch.probe_sent, conn.orch.backpressure_applied) to quickly identify problems.

Checklist:
- [x] Unit tests
- [x] Integration stress tests
- [x] Canary rollout and monitoring
- [x] SDK opt-in hooks documented (redwood-sdk-ts & python)

References:
- Linear: ENG-1467 (session lifecycle improvements), ENG-1492 (streaming backpressure design)
- Support ticket: SUP-2334 reported a production incident where ~120 sessions per host were leaking after reconnect storms.

Small illustrative snippet (server runtime change):
- session_manager.go: add grace state + probe timer
  - Before: session removed immediately on disconnect
  - After (conceptual):
    if disconnectReceived {
      session.state = GRACE
      scheduleProbe(session, config.GraceWindow)
      reducePrefetch(session)
    }

This PR intentionally keeps application-visible protocol unchanged; all knobs are opt-in per-deployment.
- Improve client disconnect/grace window handling for streaming APIs
- Introduce server idle timeout + probing ping to avoid leaked sessions
- Add flow-control/backpressure to limit in-memory KV-cache growth during slow clients
Alex Romero: Can we make the grace window configurable per-tenant?
Priya: Yes, added admin config and SDK hook; default remains 45s.
Leah Park: Are there any cases where probe could cause false positives behind aggressive NATs?
Priya: We set probe_timeout=10s which empirically avoided most NAT reordering; also probe is retried once before teardown. If we see false positives we can increase probe_timeout for affected CIDR ranges.
Samir Patel: Please ensure metrics have cardinality caps.
Priya: Metrics added are aggregated by SLO class and host; added sampling for verbose labels.
Alex Romero: Approving pending a small doc update on SDK options.
Leah Park: Approved.
Samir Patel: Approved.
