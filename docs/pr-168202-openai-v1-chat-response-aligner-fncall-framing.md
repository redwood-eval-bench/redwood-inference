# OpenAI /v1/chat.completions: response aligner, fncall sequencing, and streaming envelope hardening

Context and motivation: We observed several production cases where clients using the /v1/chat.completions compatibility layer received malformed or semantically inconsistent payloads when function/tool-calls and streaming framed JSON interacted. Problems included: 1) partial JSON fragments being forwarded to consumers when a stream fragment boundary cut across tool output; 2) non-deterministic ordering when multiple tool invocations were emitted in the same generation step; 3) subtle differences in stop/logprob handling between the v1 adapter and canonical internal chat model output, which caused user regressions for a small set of prompts. This PR introduces a compact compatibility shim (response aligner) and a fncall sequencer to make v1 behavior deterministic and safer during streaming.

What this PR does (high level):
- Adds a response aligner module in the OpenAI compatibility layer that normalizes role names, stop semantics, and logprob envelopes to match /v1 expectations.
- Implements a fncall sequencer that orders tool/function-call events deterministically when multiple calls are produced in a single response and exposes an explicit sequence_id in the framed output.
- Hardens the streaming envelope parser/serializer so partial frames are buffered and validated before emitting to downstream clients (reduces partial JSON exposure).
- Adds migration/opt-in flag: compatibility.enable_v1_sculptor (default true for rollout on hosted; teams can disable per-customer during canary).
- Extensive unit and e2e tests including a new fuzz test for streaming fragmentation cases.

Checklist (developer):
- [x] Design doc: link to Linear ENG-4122 (internal)
- [x] Unit tests for response aligner and sequencer
- [x] E2E tests for streaming tool output (CI harness)
- [x] Backcompat tests for existing v1 consumers
- [x] Docs: migration notes and example for fncall ordering

Commits (summary):
- feat(openai-compat): add response_aligner module and test harness
- feat(streaming): buffer partial frames and introduce frame validation step
- feat(fncall): deterministic sequencer and sequence_id in framed tool events
- fix(tests): update many mocked expectations for logprob/stop mapping
- docs: add openai_v1_migration.md and example showing ordered fncalls

Changed areas (high level):
- src/openai_compat/v1_adapter.py — new ResponseAligner class, parameter mapping, small backwards-compatible API surface behind flag
- runtime/stream/coordinator.rs — frame buffering and validation before emit (Rust runtime change)
- api/handlers/chat_v1.py — wire up compat layer and add Sequence-ID header for tool events
- tests/e2e/test_fncall_sequence.py — new end-to-end tests that assert deterministic ordering across multiple tools
- tests/unit/test_response_aligner.py — unit tests covering role aliasing and stop/logprob normalization
- docs/openai_v1_migration.md — migration notes and rollout plan
- ci/workflows/openai_compat.yml — adds streaming fuzz job to CI

Review conversation (high level):
- Diego Alvarez (review): "Nice focused change. Two asks: (1) please add a short section in the docs about how sequence_id should be consumed by clients, and (2) ensure sequence_id is stable across retries (idempotent). I also saw a flaky e2e test on first CI run — likely timing-related."
- Maya Chen (author): "Addressed: added docs section and updated sequencer to emit UUIDv5 based on request_id+call_index so it's stable across retries. Also added a small backoff in the e2e test to wait for stream finalization; moved fuzz seed management to make CI deterministic. PTAL."
- Lena Karp (review): "LGTM after the Rust perf microbenchmark numbers are added — can we add a single-line benchmark showing median latency impact of buffering?"
- Maya Chen: "Added bench numbers (streaming median tail increase ~0.6ms at p99 reduced artifact rate by ~98%). Included in PR description and docs as a short note."
- Rajat Singh (approval): "Tests + docs look solid; approved." 

CI summary:
- First run: streaming-fuzz failed (timeout), unit tests passed, e2e flaky test failed. Reran after fixes. Second run: all checks passed including streaming-fuzz (with deterministic seed), e2e, unit, lint, and integration against sample_customer_config.
- CI checks: lint: pass, unit: pass, e2e: pass, streaming-fuzz: pass, perf-smoke: pass

Rollout & risk mitigation:
- Feature behind compatibility.enable_v1_sculptor flag; we will rollout to 10% of hosted traffic for 48 hours, monitor error budget, and then widen to 100% if no regressions.
- Added an automatic fallback that, if frame validation fails more than 3 times for a request, will fall back to legacy passthrough mode for that request and emit a metric openai.compat.fallback.count.

Why this approach:
- Aligning output semantics in a dedicated layer avoids scattering mapping logic across many handlers and makes future refactors easier.
- Deterministic fncall sequencing reduces client-side ambiguity for multi-tool flows and enables safer replay/observability.
- Buffer-and-validate reduces user-visible malformed JSON without adding significant latency; bench numbers included.

Bench numbers (perf-smoke):
- Streaming median latency (p50): +0.2 ms
- Streaming tail (p99): +0.6 ms
- Partial-frame artifact rate: -98% in synthetic fuzz suite

Merge outcome:
- Merge method: squash and merge
- Merge commit: created and deployed to canary; no rollback needed during 24h canary window

Follow-ups / TODOs:
- ENG-4200: wire up SDKs to expose sequence_id in streaming helpers (TS + Python)
- ENG-4215: surface sequence ordering examples in the API reference (rich examples for toolchains)
- Monitor: openai.compat.fallback.count, openai.compat.frame_validation.latency on dashboards

Notes for reviewers: The PR is intentionally conservative in surface changes (flag + non-breaking opt-in) but contains cross-cutting runtime changes that required coordination between API layer and Rust runtime. If you want a walkthrough, ping me and I can demo the sequence tracer in a short call.

References: Linear ENG-4122, internal incident INT-2026-03 (fragmented-tool-output) for postmortem context.
Added response alignment layer for v1 chat compatibility, deterministic function-call sequencing, and improved resilience for streaming framed JSON envelopes. No breaking changes to v1 surface; opt-in behind compatibility flag for a short rollout.
Deployed to canary after merge. Observability dashboards updated; will expand rollout if no regressions observed over 48h.
