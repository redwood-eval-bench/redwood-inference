# OpenAI-compat output sanitizer + tokenizer boundary heuristics for stable structured streaming

Motivation: Structured-output use cases (JSON/CSV/DSL responses) remain fragile in partial/streaming modes when tokenizer boundaries or control characters are emitted mid-response. This PR introduces a conservative output sanitizer and a set of tokenizer-boundary heuristics to reduce malformed structured responses, improves OpenAI-compatibility behavior for function/tool calls during streaming, and exposes a short-circuit backend toggle for constrained decoding fallbacks.

What this changes (single-string summary): Adds a streaming sanitizer that performs low-latency, token-aware escapes and soft-truncation hints; implements a deferred-close-token heuristic to avoid emitting closing braces/quotes until an end-of-structure token is confidently produced; augments regex/CFG backends with a tokenizer-aware fallback that aligns regex anchors with token boundaries; patches the OpenAI-compat layer so function call hints are preserved and rehydrated if a streaming interruption occurs; introduces an environment toggle (OPENAI_COMPAT_CONSTRAINT_BACKEND) to prefer "token-aware-fallback" for high-integrity structured outputs.

Checklist:
- [x] Sanitizer: streaming token-level sanitizer with minimal buffering
- [x] Token boundary heuristics: conservative opening/closing emission, surrogate pair handling
- [x] Regex/CFG fallback: tokenizer-aware anchor alignment and percent-escape mapping
- [x] OpenAI compat: persist function-call hints across partial responses and replay on reconnect
- [x] Backwards-compatible defaults: feature off for older deployment profiles, opt-in via env/route config
- [x] Tests: unit + e2e streaming harness + compatibility fuzzing against existing examples

Why unique: Previous PRs addressed grammar or regex backend bugs; this work focuses on the surface between tokenizer semantics and structured output stability in streaming/function-call scenarios and ties to OpenAI compatibility behaviors. It intentionally errs on the conservative side to prioritize correctness over token throughput in affected routes.

Testing notes: Added new e2e tests run in the streaming harness (tests/streaming_structured_scenarios) that simulate mid-stream disconnects and racey token emissions. CI includes fuzz runs for token-boundary edge cases (surrogate pairs, combining marks) and regression asserts against the real-world prompt corpus used by ENG-8721.

Rollout plan: feature gated behind OPENAI_COMPAT_CONSTRAINT_BACKEND=token-aware-fallback and route-level config in console. Start with canary on dedicated cluster, monitor structured-output regressions and latency P50/P95.

Marco: Can you summarize the expected latency impact on high-throughput chat routes? Please include a P50/P95 delta from the benchmark harness.
Ava: Conservative heuristics add a small buffering window (~1-3 tokens) in streaming hotpath; measured P50 unchanged, P95 +6ms for 2048-token typical sequences (benchmarks added).
Sofie: Edge case: surrogate pair split across token boundary still produced malformed JSON in local run. Is the joiner logic handling UTF-16 surrogate pairs?
Ava: Added explicit surrogate-pair reassembly in token_boundary.rs and a fuzz test that reproduces the issue; pushed that commit.
Devon: LGTm but please add an explicit opt-out in route config for low-latency chat endpoints where strict structure is not required.
Ava: Added route-level flag preserve_structured_integrity=false and documented behavior; default remains true for OpenAI-compat constrained routes.
Adds an opt-in token-aware streaming sanitizer and tokenizer boundary heuristics to improve stability of structured outputs in streaming and function-call scenarios. Default gated behind OPENAI_COMPAT_CONSTRAINT_BACKEND; see console docs for rollout.

Impact: reduces malformed JSON/CSV outputs during mid-stream disconnects and tokenizer-edge cases; small P95 latency increase on affected routes.

Action: Enable in canary, monitor structured-output error rate and latency P95.
New route-level flag: preserve_structured_integrity (default true for compat routes)
New env toggle: OPENAI_COMPAT_CONSTRAINT_BACKEND=token-aware-fallback
Improved stability for streamed JSON/CSV/DSL outputs when using constrained decoding or function calls
This PR avoids wholesale changes to the constrained decoding API and focuses on runtime heuristics and short-term fallbacks. Future work includes a configurable integrity/latency knob surfaced in the console and tighter integration with Optimize recommendations.
