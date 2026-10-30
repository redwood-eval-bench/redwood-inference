# callsite-telemetry-envelope-and-endpoint-dimension-tags-for-api-layer

Motivation: Several production incidents and postmortems (ENG-4923) showed gaps in correlating streamed fragments, tool invocations, and short-lived worker hops to the originating API request. Existing request-id propagation was fragmented between transport layers and the OpenAI compatibility shim, which made per-endpoint metrics and the JSONL logs hard to join. This PR introduces a compact callsite telemetry envelope on the request context and surfaces endpoint dimensions as metric labels to make traces, logs, and metrics easily joinable across the stack.

What changed (high level):
- Add a lightweight TelemetryEnvelope attached to request context that contains request_id, route_id, route_version, initial_prompt_hash, and minimal client/baggage info. The envelope is intentionally small (max ~256 bytes) to avoid amplifying streaming payloads.
- Wire envelope propagation through the API transport layer, the OpenAI-compat adapter, and into tool/function-call execution paths so tool executions get the same callsite session id.
- Enrich Prometheus-style metrics with endpoint dimensions (route, variant, handler) as stable label keys. Labels are applied at instrumentation sites and exposed in per-route dashboards.
- Emit JSONL access log entries that include the envelope fields at top-level as consistent keys (request_id, route, route_version, callsite_id). This unifies previous ad-hoc keys.
- Add a small opt-in header (X-RW-Telemetry: base64(envelope)) used by Private deployments for cross-service propagation; when absent we fall back to sampled trace ids.

Backward compatibility and safety:
- TelemetryEnvelope is optional. Services that don't yet understand the header ignore it. The OpenAI-compat adapter still accepts existing request headers.
- The size and shape were constrained to avoid inflating streaming frames. The base64 header is only set for internal/restricted deployments; hosted API does not add it by default.

Testing and rollout plan:
- Unit tests covering envelope serialization, propagation to tool-execs, and label attachment.
- E2E test that streams a long response while invoking a function and asserts all emitted log lines contain the same callsite_id. Added tests under tests/e2e/telemetry_envelope_test.go.
- Canary rollout: default off for Hosted; enabled behind feature flag telemetry.envelope for Dedicated and Private. After 1 week of stable canary, flip rollout for Dedicated and enable opt-in config for Private.

Operational notes:
- Added a migration doc (docs/observability/callsite-telemetry.md) describing how to query logs and metrics for correlation patterns and how to toggle header propagation for Private installs.
- Metric cardinality: endpoint labels were designed to use stable enumerated values (route, variant, handler) to avoid high-cardinality user inputs. We reviewed Prometheus retention implications with SRE.

Why this is different from prior work: previous PRs added request ids or per-route tags in scattered places. This change is a unifying envelope plus explicit label wiring across the OpenAI shim and tool execution paths so streamed fragments, tool calls, and backend workers can be reliably joined without guessing keys.
Add a compact callsite telemetry envelope attached to every inbound request and enrich per-endpoint metric labels. No user-facing API changes. Improves traceability for streamed responses and function/tool calls.
Ethan Ruiz: Nice consolidation. Two asks: 1) Can we bound the number of baggage entries attached to the envelope? 2) Confirm the header won't be emitted for hosted customers by default.
Maya Chen: Responded - baggage is capped to 3 fields, we drop extras with a debug metric; header emission is gated by deployment mode and feature flag (see internal/config/feature_flags.go).
Priya Nair: Please run the streaming e2e twice — we've seen flakes in ci for stream_fragment_assembly. Also add a comment where prometheus labels are applied explaining cardinality choices.
Maya Chen: Added retry in e2e and expanded comment with cardinality rationale. Marked the metric label helper with a docstring.
Liam O'Connor: code LGTM after tests pass. Approving now.
CI bot: First CI run failed on tests/e2e/streaming_test.go:TestFragmentAssembly (flaky). Rerun triggered by author.
CI bot: Second CI run all checks green.
Release manager: Approved release notes wording and confirmed no migration steps required for Hosted.
Avoided changing any public SDK interfaces. OpenAI-compat adapter maps envelope request_id->x-request-id for legacy consumers. Private installer docs updated to show how to enable header propagation.
Query logs: grep for callsite_id in JSONL access logs and ensure a single callsite_id appears across streamed fragments and function call logs.
Prometheus: use label 'route' and 'variant' in per-endpoint dashboards to verify metrics are attributed correctly.
E2E: run tests/e2e/telemetry_envelope_test.go -- it asserts envelope present in tool execution logs.
