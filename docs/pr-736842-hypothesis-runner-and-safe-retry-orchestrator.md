# Hypothesis-driven diagnostic runner and safe-retry orchestrator

Motivation: On-call responders need quicker, lower-noise triage. Existing probes run a fixed battery which is often noisy, expensive, and yields too many false leads. This PR introduces a hypothesis-driven diagnostic runner that prioritizes lightweight speculative probes derived from alert context and trace snippets, and a safe-retry orchestrator that governs re-run behaviour for higher-cost probes to avoid alert storming and flapping.

Design summary: The system ingests alert payloads and short trace/metrics extracts, scores potential failure hypotheses using a compact feature extractor, then schedules a tiered probe plan: cheap inference probes first (fingerprint, prefix KV cache check, config fetch), followed by targeted stateful checks (TCP/HTTP/DB quick checks) only if the hypothesis score warrants. If a follow-up probe is high-cost (long tail or heavy API queries), the safe-retry orchestrator enforces exponential backoff, rate caps per-alert, and a small circuit breaker window so probes cannot overload downstream services. All runs produce a compact diagnostic artifact that can be attached to the incident card.

What changed (high level): Added a new hypothesis scoring package, new probe adapters for elasticsearch/tcp/postgres quick-checks, a scheduler that emits tiered plans, and a lightweight artifact serializer. New operator implements safe retry/backoff/circuit semantics plus telemetry of probe outcomes to avoid probe amplification. Hooked into incident-bot's existing runbook attachment flow and added new admin flags to control global probe budgets.

Checklist: - [x] Core hypothesis scorer and runner implemented - [x] Probe adapters (es/tcp/postgres/http) added - [x] Safe-retry orchestration with backoff/circuit implemented - [x] Unit tests for scorer, scheduler and retry operator - [x] End-to-end local harness + mock services - [x] Docs and admin config entries - [x] Observability: metrics (probe_count, probe_cost, probe_failure_rate), traces and per-alert probe logs

Testing notes: Local e2e harness simulates alert types seen in SRE-67; CI initially failed due to a flaky test that relied on wallclock timings for backoff — fixed by moving to a mockable clock in retry operator. Load test (small-scale) showed probe budget enforcement prevents >3x amplification during alert bursts.

Rollback plan: Feature guarded behind runtime flag (enable_hypothesis_runner) and probe budgets. If we observe unexpected amplification, ops can toggle the flag to disable the runner immediately; safe-retry has default conservative limits to avoid escalation.

Privacy/security: Probes follow existing data sanitization paths; the artifact serializer redacts PII and secrets before attaching to incident cards. New adapters authenticate using existing service account mechanism.

Related: Built to complement automated runbooks and the compact snapshot format introduced in ENG-1712. Ties into SRE-67 for prioritized probe types.

Merge notes: Addressed reviewer feedback around logging verbosity and added a small metrics tag for probe_origin (hypothesis|manual). Squash-merged to main on 2026-02-03.
Evan Clarke: Overall looks good. Can the scorer be tuned without a deploy? Consider exposing a weight table via config map.
Priya Ramesh: Good idea — added a runtime-config path and documented override semantics. Also added validation to prevent negative weights.
Marta Silva: The safe-retry defaults are conservative but please ensure we surface probe budget exhaustion in a visible metric/alert.
Priya Ramesh: Added probe_budget_exhausted counter and an ops dashboard note in the docs.
Tobias Nguyen: Nit: reduce debug logging for successful cheap probes at INFO -> DEBUG to cut noise in busy clusters.
Priya Ramesh: Reduced verbosity and added a sample rate limiter for repeated identical probe logs.
Evan Clarke: CI red on first run due to TestRetryTiming; I re-ran CI after the mock clock change and it passed — approving.
Marta Silva: Approved after verifying metrics and a quick e2e run on staging.
Repository Admin: Squash and merge — merge commit created by Priya Ramesh on 2026-02-03.
CI: initial run failed (TestRetryTiming flaky) -> fix with mock clock -> re-run CI passed (build, unit, e2e-harness). Coverage: +2.3% in internal/retry. Lint: passed. Security scan: no high findings.
- Add hypothesis-driven diagnostic runner to prioritize lightweight probes and reduce noisy triage.
- Introduce safe-retry orchestrator to enforce backoff, per-alert rate caps, and circuit breaker semantics for high-cost probes.
- New probe adapters: elasticsearch, tcp, postgres (light checks).
- New observability metrics: probe_count, probe_cost, probe_failure_rate, probe_budget_exhausted.
Deploy to staging behind runtime flag and run synthetic alerts from SRE-67
Create ops dashboard with probe_budget_exhausted and probe_cost histograms
Schedule a 1-week monitoring window before enabling in production
