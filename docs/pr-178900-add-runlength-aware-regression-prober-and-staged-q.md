# Add run-length-aware regression prober and staged quarantine gate

Motivation: We frequently see noisy single-bucket spikes that trigger brittle canary decisions and, separately, slowly growing sustained degradations that evade pointwise detectors. This PR introduces a run-length-aware regression prober plus a staged quarantine gate to distinguish transient noise from sustained regressions and to throttle traffic gracefully when a persistent problem is detected.

Design summary: The prober computes run-length statistics over discretized metric buckets (default 1-minute buckets) using a lightweight run-length encoding heuristic: it tracks consecutive buckets above configurable anomaly thresholds and reports a run-length score (RLS) per cohort. RLS is combined with existing percentile and throughput fingerprints to surface sustained degradation signals. The staged quarantine gate consumes RLS and applies a tiered isolation policy: (1) advisory scoring only (no traffic change), (2) soft quarantine (route X% of new traffic to fallback model), (3) hard quarantine (steer all new traffic off the candidate) with cooldown budgets and circuit-breaker style backoffs.

Key goals: reduce false positives from isolated spikes; detect and act on sustained regressions faster; provide predictable, auditable staged rollbacks with quarantine budgets and automatic decay.

What changed (high level): implemented prober runtime module perf_canary/probers/runlength_prober.py (core logic + parameters), added controller hook perf_canary/gates/staged_quarantine.py, exposed two new metrics (pc_runlength_score and pc_quarantine_budget_usage), added unit & integration tests in perf_canary/tests/test_runlength_prober.py and perf_canary/tests/integration_test_quarantine_flow.py, and updated docs at docs/canaries/runlength-prober.md with configuration examples and recommended thresholds.

Operational notes: default configuration is conservative: RLS threshold tuned to detect >=3 consecutive anomalous buckets at p95+ tail regressions. Quarantine budgets are per-tenant and persist for 24h with exponential cooldown on repeated activations. The gate logs all decisions to the canary audit stream (audit.canary.decisions) for post-mortem.

Rollout plan: behind feature flag perf_canary.runlength_prober.enabled (off by default). Start with synthetic probes and shadow traffic for 48h, enable advisory scoring for canaries, then enable soft quarantine for low-risk services, gradually widen.

Related: builds on ENG-8421 (cohort fingerprint improvements) and ENG-8470 (metric window autosizing). SUP-2287 reported a false-positive incident on 2026-02-05 that motivated run-length logic.

Notes on evaluation: new prober reduces alert churn in stress tests (see integration test results attached in CI) while catching multi-minute slowdowns in end-to-end replay scenarios.

Detailed configuration examples and thresholds are included in the docs file and config defaults in perf_canary/config/defaults.yaml.
Marco Chen (requested changes) 2026-02-22: Mostly LGTM. Please rename metric pc_runlength_score -> pc_rls to match our existing short metric naming and add a short comment explaining the exponential cooldown math. Also expand the integration test to assert budget decay after a hard quarantine.
Aisha Patel (author reply) 2026-02-23: Addressed metric rename and added comments in staged_quarantine.py. Integration test extended to assert budget decay. PTAL.
Sven Müller (approved) 2026-02-25: Verified behavior on replay environment; soft quarantine flow worked as expected and advisory mode produced zero traffic shift. Approving after CI fixes.
Priya Rao (approved) 2026-02-28: Code review: concurrency model looks safe; left one minor doc nit which was fixed.
Marco Chen (approved & merged) 2026-03-01: All requested changes addressed. Squash-merged with release note entry.
Introduce run-length-aware canary prober (pc_rls) and a staged quarantine gate with per-tenant budgets. Off by default behind feature flag; reduces false positives from single-bucket spikes and enables controlled staged rollback for sustained regressions.
