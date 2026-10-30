# introduce-frugal-threshold-tuner-cli-for-batching-and-cache-defaults

Motivation: Production workloads show highly variable per-request sequence length and cold-KV cache hit patterns; teams repeatedly tune batch and cache thresholds by hand. This PR introduces a small, opinionated CLI that runs a fast local probe against a workload sample and suggests cost-conscious defaults for continuous batching, prefix/KV cache activation thresholds, and conservative quantization profile hints. The goal is to reduce unit token cost while respecting 99th-percentile latency SLOs and avoiding quality regressions.

What this PR adds: a new CLI command (redwood optimize frugal-tune) that: 1) accepts a workload trace (JSONL of request metadata) or pulls a sampled profile from the console API, 2) runs a lightweight simulator that models batching latency curves, cache hit probability by prefix length, and quantized model throughput deltas, and 3) emits a recommended defaults bundle (batch_size_hint, batch_timeout_ms, cache_min_prefix_tokens, preferred_quant_profile) plus an annotated report with economic tradeoffs (estimated cost per 1M tokens, expected p99 latency impact). The CLI is intentionally conservative: recommendations include a safety margin and explicit rollback instructions.

Technical approach & constraints: The simulator uses the runtime's empirical latency models (extracted from existing perf profiles) combined with a compact queuing model to estimate tail latency at different batching shapes. Cache heuristics are derived from sampled prefix reuse distributions. Quantization guidance is based on precomputed quality-versus-throughput curves for supported model families; the CLI will only recommend a lower-precision profile when estimated token-quality loss is within a user-provided tolerance flag. The tool is offline-only and does not modify live configs; output is a JSON bundle compatible with the platform defaults API.

Changeset summary: adds a new small Go binary under tools/opt-cli/, a simulator package under internal/econsim/, a README and quickstart in docs/opt/, unit tests covering simulation edge cases, and an integration test that validates full CLI flow against a synthetic trace. Also adds small telemetry events (opt_tune.run, opt_tune.apply) gated behind an opt-in flag.

Checklist:
- [x] design doc reviewed (ENG-3219)
- [x] unit and integration tests
- [x] CLI help and README
- [x] conservative defaults and rollback instructions
- [x] safety telemetry (opt-in)

Usage example: redwood optimize frugal-tune --trace sample.jsonl --max-quality-drop 0.01 --out recommended-defaults.json

Risks and mitigation: Incorrect modeling could suggest aggressive defaults that violate latency SLOs. Mitigations: 1) CLI includes p99 latency estimates and flags when suggestion risk is high; 2) recommendations include explicit canary rollout steps; 3) recommendations default to no-op when sample size is too small.

Follow-ups: Add an optional dry-run mode that can replay a small subset of live traffic against suggested defaults in sandboxed Dedicated capacity (future work, ENG-4207).
Diego Martinez: Overall looks good. Can we make the quantization quality model pluggable so it can be swapped when we onboard new families?
Aisha Clarke (author): Done — added a registry interface and a simple JSON loader for family curves. See internal/econsim/models.go
Ravi Patel: CI flake on integration test observed (TestEndToEnd) — appears to be timing-related, re-ran and green.
Lena Sørensen: Please add a short note in docs about privacy considerations when sending traces to the Console API (opt-in).
Aisha Clarke: Added privacy note and telephone-home opt-in flag to the README
Added 'redwood optimize frugal-tune' CLI to generate conservative batching, cache, and quantization defaults from workload traces. CLI is opt-in and provides safety guidance for canarying recommendations.
Telemetry events are opt-in by default. If you enable them for support diagnostics, expect light-volume events: opt_tune.run and opt_tune.apply. No automatic config writes performed by the CLI.
