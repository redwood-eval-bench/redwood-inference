# Graceful schema rollout: beta opt-in header and legacy-response adapter

Motivation: We need a non-breaking, observable path to roll clients from the v1 request/response schema to the new v2 schema (structured output + stable function/tool calling semantics + streaming framing changes). This PR introduces a beta opt-in header, a deprecation warning header for v1, and a lightweight legacy-response adapter that rewrites v2 streaming frames to v1-compatible shapes for clients that have not yet opted in.

What this PR does (high level):
- Adds support for a request header X-Redwood-API-Schema-Beta: v2 to opt into the new schema. When present, server will emit v2-shaped responses and streaming frames.
- Adds a response header X-Redwood-API-Schema-Warning when the client used the implicit (no-header) v1 path: value set to "v1-deprecated; remove by 2026-06-01". This makes deprecation signals machine-readable.
- Implements a legacy-response adapter middleware that detects v1 clients (no beta header + compatibility token) and rewrites streaming delta frames and final structured payloads into v1-compatible shapes. The adapter is intentionally conservative: it does not attempt to reconstruct missing function-calling traces; instead it adds a best-effort fallback field ("function_calls_approx") and emits a deterministic warning event to the stream.
- Adds end-to-end tests and local canary toggle so we can roll this to 10/50/100% of traffic via feature flags.

Design notes and rationale:
- Opt-in must be explicit: we avoid silent breaking changes. The X-Redwood-API-Schema-Beta header gives fast adopters immediate access to v2 while leaving defaults stable.
- The legacy-response adapter reduces churn for slow-updating clients but intentionally marks adapted responses as deprecated to encourage migration.
- Streaming rewrite approach: for token-delta frames that include structured fragments, adapter merges fragments into a flat text delta with an additional metadata field encoded as JSON string to preserve structure for downstream parsers.

Testing and verification:
- Unit tests for header parsing, adapter framing, and warning header emission.
- Integration tests that run a small model shim and assert stream shapes for v1 (no header), v2 (beta header), and adapted streams.
- Manual validation performed against the TS and Python SDK examples.

Rollout plan (ops):
- Deploy behind a feature-flagged rollout: 10% for 24h, monitor error/rate metrics and structured-output regressions. If stable, increase to 50% for 48h, then 100%.
- Alerting: added a new SLO-oriented dashboard for schema-mix ratio and adaptation rate.

Checklist:
- [x] API header handling implemented
- [x] Adapter middleware implemented with tests
- [x] E2E integration tests added
- [x] Docs updated (openai_compat.md) with migration guidance and header examples
- [x] Rollout dashboard and feature flag gating configured

Notes for reviewers:
- Look closely at the streaming rewrite code (adapter/stream_rewrite.go). The rewrite is intentionally conservative to avoid inventing behavior. If you find a client scenario where data would be lost, call it out and propose a specific migration note.
- I included a short example of header usage in the docs: "curl -H 'X-Redwood-API-Schema-Beta: v2' ..."

Related: This complements ENG-4701 (structured-output eval harness) and ties into the Dedicated rollout work in ENG-4642.
Ethan Cole: Overall looks good. Request: make the warning header name stable (suggest X-Redwood-API-Schema-Warning) and include an RFC3339 timestamp in the header value so we can correlate warnings with deployments.
Sofia Martinez (author): Addressed in b28c3e2 — header now includes timestamp. Added tests asserting header pattern.
Priya Nair: Adapter logic seems fine, but I'm worried about performance impact on high-throughput routes. Can we short-circuit the adapter for requests > 2k tokens?
Sofia Martinez: Added a short-circuit check in middleware to bypass adapter for long-context requests (9bd4f2c) and instrumented metric compat.adapter_bypass_count.
Luca Romano: Tests flaky on CI initially (stream tests). We should add a retry wrapper and mock timing to stabilize.
Sofia Martinez: Added retry and reduced wallclock-sensitive timing in tests (7e2b1a3). CI green locally.
Ethan Cole (approval): Approved-after-changes — nice work. Be sure team oncall knows the rollback flag path.
Release Manager (comment): Merging with squash. Please add a short note to release notes about the deprecation header so client teams are aware.
Introduces an opt-in header (X-Redwood-API-Schema-Beta: v2) to enable the v2 request/response schema. Adds a deprecation warning header for legacy clients (X-Redwood-API-Schema-Warning) and a conservative legacy-response adapter for streaming and structured output compatibility. Rollout will be feature-flagged; monitor the schema-mix dashboard and migration guide in docs/openai_compat.md.
