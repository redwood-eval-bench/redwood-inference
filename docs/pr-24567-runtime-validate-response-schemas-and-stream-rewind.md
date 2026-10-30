# runtime-validate-response-schemas-and-stream-rewind

Adds a runtime-level JSON Schema validator for response_format-based structured output and a streaming rewind+retry path for OpenAI-compat streaming. Motivation: users running in strict schema mode need deterministic validation errors while still benefiting from streaming; previously a schema failure in the middle of a stream simply produced a truncated or invalid payload and an opaque error. This PR implements a low-overhead validator that runs on streamed chunks, detects schema divergence as early as possible, provides a compact machine-readable validation_failures payload, and when configured to 'rewind_on_schema_failure' will rewind the stream to the last safe token boundary, request an on-the-fly correction from the model (via a short re-prompt), and replay a corrected substream to the client. Changes include: runtime validator, OpenAI-compat transport hooks, SDK helpers for Python/TS, docs + examples, and end-to-end tests. Checklist: - [x] implement streaming validator with incremental checks - [x] add rewind+retry mechanism with backoff & max_retries config - [x] expose response_format validation errors to API clients (structured) - [x] add SDK ergonomics (helpers to set strict_mode, rewind_on_schema_failure) - [x] update standard OpenAI-compat behavior to return structured validation metadata when compatibility mode is enabled - [x] add unit/integration tests and docs examples - [x] safety review for prompt-repair flow Links: Linear: ENG-3120, Docs PR: #1327 (external), Design note: https://linear.app/redwood/notes/ENG-3120/streaming-schema-rewind (internal). This description is intentionally continuous and compact.
Marco Li (review): Can you clarify the performance impact on 8k context models? Suggest a microbenchmark and a config knob to disable streaming validation on high-throughput routes.
Aisha Patel (author): Added a microbenchmark in tests/benchmarks and added 'validation_enabled' per-route config. See serving/runtime/schema_validator.go for fast-path checks that avoid allocations when disabled.
Rima Das (review): We're exposing a structured 'validation_failures' field to clients. Need to ensure it doesn't leak internal tokens in PII-sensitive flows.
Aisha Patel (author): Validation metadata is redacted by default and can emit a high-level enum + field path. Full error details are gated behind enterprise telemetry and only returned when 'expose_validation_details' is enabled (opt-in for Private/Dedicated customers).
Evan Thompson (review): Tests look good. Please add a migration note for existing OpenAI-compat users; the compatibility transport changes headers.
Aisha Patel (author): Added docs/guides/structured-output.md migration section and the SDK helpers to automatically adapt existing calls.
// incremental validation loop (pseudocode)
for chunk in stream {
  accumulate(chunk)
  if schema_validator.check_prefix(accumulated) == FAIL {
    emit(validation_failures)
    if cfg.rewind_on_schema_failure {
      rewind_to_safe_boundary()
      request_repair_subprompt()
      replay_corrected_substream()
    } else {
      close_with_error()
    }
  }
}
// openai_compat header added
X-Redwood-Validation-Meta: {"mode":"strict","status":"ok|error"}
// SDK helper (typescript)
export function withStrictSchema(client, schema, opts = {}) {
  return client.with({ response_format: { schema, strict: true, rewind_on_schema_failure: opts.rewind } })
}
Add runtime streaming JSON Schema validation with optional rewind+retry behavior. New API knobs: response_format.strict (boolean), response_format.rewind_on_schema_failure (boolean). Structured validation failures are returned in OpenAI-compat responses under 'validation_failures' when enabled. See docs/guides/structured-output.md for migration steps.
Risk: additional CPU on streaming paths for prefix validation. Mitigation: validation is opt-in per-route, fast-path guard to skip allocation when disabled, and a config to only validate on sampled requests. Rewind path limited to small correction window and capped retries to avoid increased tail latency.
New unit tests for incremental validator, integration tests that repro schema divergence mid-stream, and e2e scenario that exercises rewind+repair against a mocked LLM. Benchmarks show ~3-7% cpu increase when validation is enabled on 4xA10 routes; disabled-by-default for high-throughput tiers.
prompt-repair flow reviewed by security. Repair prompts are templated and sanitized; no raw client prompt content is appended to repair requests. Audit log entries added for any automatic repair actions.
Feature gated: enable on Hosted API for a limited set of customers for the initial rollout. Notify SRE on rollout to Dedicated clusters to watch tail latency. Docs and SDK releases must land alongside this merge.
Added detailed guide: docs/guides/structured-output.md, updated OpenAI-compat migration notes and SDK example snippets in python and typescript.
examples/structured-output/rewind_example.py: demonstrates strict mode with rewind_on_schema_failure and how to parse validation_failures
examples/structured-output/rewind_example.ts: TS SDK example that toggles strict mode and shows client-side fallback
