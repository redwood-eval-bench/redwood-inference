# resumable-session-broker-and-soft-timeout-reconnect-protocol

Context: Customer and internal workloads continue to surface issues where client disconnects during streaming can either lose inflight tool/function calls or leave server-side resources pinned until hard TTL expiry. This causes tail latency spikes for other tenants and operator toil responding to stale sessions. We need a lightweight server-side session broker that centralizes resume tokens and coordinated soft timeouts so the server and client share a deterministic window for reconnect/reconciliation.

What this PR does: Adds a resumable session broker and coordinated soft-timeout/reconnect protocol for streaming RPCs (OpenAI-compatible delta stream). The broker lives in the streaming control plane and holds minimal session metadata: resume_token -> (last_ack_offset, lease_expiry, in_flight_tools). The runtime uses the broker to: 1) convert abrupt TCP/WS disconnects into a 30s soft-quarantine window (configurable per-customer), 2) allow clients to rejoin by presenting a resume token and last_acked offset (server replays suffix of buffered events + any pending tool call confirmations), 3) evict sessions deterministically after coordinator-enforced TTLs and graceful aborts.

Compatibility and safety: To preserve OpenAI-compat behavior we keep the existing event stream envelope ("delta"/"done"/"tool_call") but add optional meta events ("resume-offer", "resume-accepted", "soft-abort") that client libraries can ignore if unaware. Tool/function calling is hardened by introducing a lightweight idempotency token (stream_call_id) carried in tool_call events; the broker tracks in_flight_tools to suppress duplicates on resume. We also add server-side confirmation messages for toolcall completion so clients that rejoin can reconcile side-effects.

Operational controls: Admins can tune soft_quarantine_seconds, resume_window_seconds, and broker_eviction_policy. We expose per-customer metrics: resume_attempts, resume_success, soft_evictions, orphaned_tool_calls, and broker_cache_size. Default policy: soft_quarantine_seconds=30, resume_window_seconds=60.

Testing and rollout: Unit and integration tests added for race windows (disconnect while tool_call inflight, client resumes after partial delta stream). E2E in perf-canary shows median p99 disruption for a minor abort scenario reduced from ~550ms to ~120ms with negligible throughput impact. We feature-flag the broker behind runtime feature gate `stream_resume_broker_v1` and rollout to a subset of production clusters for 48h canary prior to global enable.

Notes: This PR intentionally keeps replay buffers bounded (server-side buffer limited by bytes_per_session and time window) and does not change model execution semantics. For very long lived sessions customers should use dedicated capacity or explicit checkpointing. See linked Linear items for follow-ups (auto-reconcile evaluator sets, audit log enrichment).
Jonah Li: We need to ensure the broker store scales - can we shard by customer id?
Asha: Broker instances are sharded by customer in the design doc; added sharding comments and bench in tests (see broker.rs).
Priya Kapoor: How do we avoid duplicate side-effects from tool calls on resume?
Asha: Introduced stream_call_id and server confirmations; broker suppresses replayed tool calls with same id (see toolcall.rs).
Marcus Feld: CI flake on integration test - intermittent timing issue observed.
Asha: Tightened clock mocking and added jitter tolerances; reran CI and fixed flake.
Add resumable streaming session broker and coordinated soft-timeout/reconnect protocol. Introduces resume tokens, stream_call_id for idempotent tool calls, and server-side soft quarantine windows. Feature-flagged behind stream_resume_broker_v1.
