# anticipatory thin-client kv prefill orchestrator and pipeline watermarking

Motivation: Interactive, low-context workloads suffer from latency variance when decode and prefill stages contend for KV priming and kernel slots. Prior work focused on chunking and decode overlap but we observed cases where clients over-prime KV entries or the runtime prematurely stalls decode to wait for large prefill batches. This PR adds an anticipatory orchestrator that coordinates a thin-client handshake to pre-warm KV entries only for candidate micro-slices and enables pipeline watermarking so prefill and decode stages can overlap without causing kv overfill or excessive kernel churn.

What this change does (single-string summary): Implements an in-runtime anticipatory prefill coordinator, a low-overhead thin-client handshake protocol to convey likely continuation lengths, a micro-slice scoring estimator to choose which prefixes to pre-warm, and a watermark-based overlap controller that throttles prefill inflight tokens rather than whole requests. The orchestrator is opt-in via runtime config prefill.anticipatory_kv=false|true and experimental knobs for estimator aggressiveness.

Key design points: 1) Thin-client handshake: small extra header from SDKs that includes predicted continuation distribution (median/90p/99p in token units) and a request-class hint. If absent, orchestrator falls back to default heuristics. 2) Micro-slice scoring: estimates marginal benefit of priming a slice vs expected decode latency; prioritizes slices with high expected reuse and low priming cost. 3) Watermarking: rather than blocking decode when a prefill batch is in flight, we maintain token-level watermarks per GPU-worker—prefill can run until soft watermark then yields kernel slots to decoder. 4) Soft eviction: a lightweight KV tag that allows non-critical prefixes to be evicted earlier under memory pressure.

Compatibility: No public API schema changes beyond optional header; SDK patches (redwood-sdk-go and -python) included in this PR to demonstrate end-to-end handshake. All changes are behind a feature flag and safe to roll back.

Benchmarks: Microbench harness (interactive chat, average prompt length 24 tokens) on G4dn-like nodes: - p95 latency: -14% (from 78ms -> 67ms) - p99 latency: -22% (from 220ms -> 171ms) - throughput (tokens/sec): +12% for 8-concurrent clients under realistic arrival jitter. Memory impact: average KV resident entries increased ~5% during warm windows but steady-state unchanged due to soft-evict.

Tests and validation: Unit tests for micro-slice scorer and watermark controller; end-to-end integration test that runs 1000 simulated short chats; canary deployment run against perf-canary for 48 hours with no production regressions. Load tests attached under bench/interactive/bench_anticipatory.yml.

Files changed (high-level): runtime/prefill/anticipatory_orchestrator.cpp - new orchestrator logic runtime/prefill/watermark_controller.h/.cpp - watermarking and throttling primitives runtime/kv/soft_tag_eviction.h/.cpp - KV tagging and eviction policy sdk/python/redwood_handshake.py - optional thin-client header helper sdk/go/handshake.go - same for Go tests/prefill/test_anticipatory.py - unit+integration tests docs/notes/anticipatory-prefill.md - design notes bench/interactive/bench_anticipatory.yml - bench config

Commit summary: [c1f6a7e] initial orchestrator skeleton and interfaces; [d2a4c98] micro-slice scorer and heuristics; [e19b5aa] watermark controller and unit tests; [f4b3c22] SDK handshake helpers and integration; [a70d11b] bench configs and docs; [b3e9f66] tests/flakiness fixes and CI tweaks.

Checklist: - [x] Design doc and perf plan - [x] Unit tests - [x] Integration tests - [x] SDK minimal changes - [x] Benchmarks run - [x] Feature flag gating - [x] Canaried on perf-canary

Notes: The estimator uses a small exponential-moving-average state per client to avoid being misled by one-off long continuations. We keep the handshake payload intentionally compact (3 numbers + class token) to avoid increasing request size. Future work: integrate with the accelerator-level kernel priorityer to reduce kernel context switches when orchestrator yields.

Related: Builds on ideas from ENG-6419 (stage overlap) and ENG-7113 (kv admission policies). Linked support ticket SUP-1983 reported elevated p99 on thin-client mobile workloads which motivated the thin handshake.

Risks and mitigations: Overly aggressive prefill can waste KV memory; mitigated by soft eviction and conservative defaults. SDK handshake is opt-in so clients won’t be forced into new behavior.

Merge notes: squashed into main; feature flag remains off by default.

Introduces an optional anticipatory KV prefill orchestrator and lightweight thin-client handshake to reduce p99 tail latency for interactive workloads. Feature is behind runtime flag prefill.anticipatory_kv and off by default.
Diego Velasquez: Nice design. A few questions: (1) how do we ensure the handshake can't be spoofed or abused by clients? (2) can the watermark controller be configured per-tenant?
Priya Rao: Reply: handshake is authenticated with existing request signature; we only use it as hint not as authority. Watermarks are runtime-configurable and we expose per-tenant overrides in Dedicated, default off in Hosted.
Marta Chen: Please add a unit test that simulates frequent long continuations to validate the EMA guards. Also note: platform-team asked that any SDK header be backwards-compatible with older clients (no hard failure).
Priya Rao: Added EMA edge-case test and SDK helpers return empty header if not supported by client; updated tests accordingly.
Noah King: CI initially failed on flakey bench harness. After retries and a small timing adjustment the pipeline is green. Approving pending documentation tweak.
Marta Chen: Approved after docs update.
