# response negotiation: fine-grained jsonschema error modelling and openai-compat mapping

Summary: Adds a negotiated response_format/strictness model for JSON Schema validation on the OpenAI-compat and Redwood API surfaces. This PR introduces a multi-mode validator (permissive, strict, diagnostic) and a mapping layer so schema validation failures are exposed as structured OpenAI-compatible errors where configured. Motivation: customers have asked for deterministic error semantics when using structured output + streaming + function/tool calling. Previously failures during streaming were surfaced as opaque stream truncation or server errors which made client-side retries and tool-invocation unreliable. This change makes the behavior explicit and configurable.

What changed (high level):
- New response_format field: {"type": "json_schema", "schema": <uri/base64>, "mode": "permissive|strict|diagnostic"} (see examples). The runtime negotiates the validator mode with the OpenAI-compat shim.
- Validator modes: "permissive" (best-effort correction and warnings inline), "strict" (on first unrecoverable schema violation: emit structured error and close stream), "diagnostic" (like strict but includes full diff and canonicalized candidate in the final event payload).
- Streaming behavior: validators are incremental and run as tokens arrive. For strict mode we avoid mid-token aborts by finishing the current token/chunk, emitting an error event that matches OpenAI's "invalid_request_error" shape, and then cleanly terminating the stream with a final JSON envelope so clients can rely on deterministic EOF semantics.
- Function/tool calling integration: if the model attempts to emit a tool call that violates the declared output schema, we intercept and convert to an explicit FunctionCallValidation error (OpenAI-compat code: function_call_error). In permissive mode we will instead attempt small repairs (close missing JSON, add placeholder typed fields) and emit a x-redwood-schema-warning header.
- Error mapping: adds a compatibility layer to translate our internal SchemaError objects into OpenAI-compatible error responses (error.type, error.message, error.param, error.code). This is configurable per-tenant.
- Observability: new metrics (schema.validation.mode.counter, schema.validation.failure_reason, schema.repaired_count) and tracing spans (validator.incremental, validator.finalize) so we can correlate validation failures with latency/cost.
- API compatibility: OpenAI-compat clients can opt-in using the response_format shim. Default behavior remains unchanged for existing users.

Files/areas changed: (high-level list)
- runtime/validator/incremental_validator.go - core incremental JSON Schema runner + patcher hooks
- server/handlers/openai_compat.go - negotiation and error mapping layer
- server/handlers/streaming_writer.go - emit structured error envelope and finalize stream semantics
- telemetry/metrics.go - new metrics and tag propagation
- sdk/python/openai_compat.py - small helper for response_format construction and examples
- tests/e2e/structured_output - new test matrix for streaming + function calls + schema modes

Commits summary: single-author work with iterative testing and a follow-up patch that tightened the final stream envelope. High-level commit messages: 
- initial incremental validator and mode switch scaffolding
- wire up OpenAI-compat negotiation and compatibility mapping
- streaming finalization + deterministic EOF for strict/diagnostic modes
- function call interception and repair heuristics
- add metrics, tracing and tests
- address review feedback: clearer error codes and docs

Backward compatibility notes:
- No change to existing default behavior unless response_format.json_schema is provided by the client.
- Tenants can opt into strict behavior at per-route or per-API-key level via console flags in a follow-up UI PR (PM-571).

Testing:
- New unit tests covering incremental schema acceptance, mid-stream violation, and repair heuristics.
- E2E harness runs against OpenAI-compat shim to ensure error shapes match integration tests.
- Manual replay of incident trace SUP-431 confirmed that previously observed stream-truncation is now surfaced as a FunctionCallValidation error and mapped appropriately.

Rollout plan: Merge gated by canary rollout: enable as opt-in default=false for hosted; enable for Dedicated beta customers by config. Monitor schema.validation.failure_reason and schema.repaired_count during canary.

Notes/limitations: The validator currently supports JSON Schema draft-07 features we rely on; advanced features (examples, remote $ref resolution with non-whitelisted hosts) are disabled for safety. Diagnostic payloads may reveal user content and are thus only emitted if tenant config allows "include_diagnostic_payloads".

Related: ENG-1289 (server-side validator effort), PM-571 (product opt-in UX), SUP-431 (customer incident that motivated deterministic errors).
Diego Alvarez (reviewer): Can we avoid emitting PII in diagnostic mode by default? Asha: Diagnostic mode is gated behind tenant config 'include_diagnostic_payloads'. We added a defensive scrubber to the finalize step.
Priya Shah (reviewer): How does this interact with batching? Could one request's schema failure block the batch? Asha: Validator runs per-request; batching kept at token-level scheduling. We added a cancellation token for validator work so other requests in batch are unaffected.
Michael Grover (reviewer): The OpenAI mapping must preserve error.code for our enterprise contracts. Request: include a mapping table in docs. Asha: Added mapping table to docs/api/response_format.md and included unit tests asserting shape equivalence.
QA (automated): flaky e2e timed out on first run (test_strict_mode_stream) — reran pipeline and all checks passed.
Author response to feedback: tightened error messages, added metric tags for tenant_id, and reduced diagnostic payload size.
Approval: Diego approved after doc update; Priya approved after batching clarification; Michael approved after mapping table addition.
Add: response_format json_schema negotiation with strict/permissive/diagnostic modes, structured OpenAI-compatible schema error mapping, streaming-safe finalization semantics, and function-call validation/repair heuristics.
