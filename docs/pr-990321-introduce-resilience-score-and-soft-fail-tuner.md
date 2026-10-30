# Introduce resilience score and soft-fail tuner for prompt-set regression gating

Motivation: Current eval CI gates are binary and sensitive to noisy prompt-level volatility. Teams have asked for a way to distinguish transient/flaky prompt regressions from systemic quality regressions and to tune gate sensitivity without creating a flood of hard failures. This PR introduces two components: (1) a resilience_score metric that summarizes historical volatility and recovery behavior per prompt cohort, and (2) a soft-fail tuner that converts hard CI gate failures into graduated outcomes (soft-fail / warn / hard-fail) based on resilience_score and run-level variance.

What this changes: Adds a new resilience index calculation to the eval metrics pipeline, updates the CI gate decision logic to consult resilience thresholds, and introduces a lightweight standalone CLI (evalctl resilience simulate) for teams to test gate sensitivity locally. The soft-fail tuner also hooks into the existing alerting pipeline so that low-severity regressions create slack/webhook warnings instead of PagerDuty pages by default.

Files and surface area: - metrics/resilience_index.py: implements resilience_score calculation and smoothing; includes configurable half-life and volatility window. - ci/gates/soft_fail_tuner.py: gate policy that maps metric delta + resilience to outcome. - evalctl/commands.py: adds 'resilience simulate' command to produce synthetic historical traces and run decisions. - tests/*: unit and integration tests for scoring, gating, and CLI. - docs/ops/resilience-guide.md: operational guidance and recommended thresholds.

Design notes: resilience_score is a bounded 0..1 score; low values indicate historically unstable prompts (high volatility), high values indicate stable prompts. The score combines: (a) smoothed historical variance of the prompt metric, (b) the median time-to-recover after previous regressions, and (c) prompt-level coverage (sample count). We use a rank-based normalization to make the score robust across metrics and models.

Backward compatibility: Default thresholds produce the previous behavior (hard-fail on threshold breach). Teams can opt-in to soft-fail via per-repo CI config, or run the CLI to preview effects. The metric calculation uses existing stored eval traces; rolling back is safe (no data schema changes).

Checklist:
- [x] resilience score implementation with tests
- [x] soft-fail gate integration in CI
- [x] CLI for simulation and local tuning
- [x] docs and runbook
- [x] e2e smoke test in CI

Related: ENG-1824 (prompt-level regression noise reduction); SUP-743 (customer asking for fewer noisy pager incidents).
Diego Martinez: Nice direction. Can you explain how the half-life defaults were chosen and whether teams should tune them?
Arielle Chen: Answered: defaults are 7 days half-life based on cohort experiments; added flags to config and CLI so teams can adjust per-prompt cadence.
Priya Kapoor: Concern: soft-fail might hide real regressions if thresholds are set too loose. Recommend an initial alerting policy that escalates repeated soft-fails after 3 occurrences within 2 weeks.
Arielle Chen: Implemented escalation by adding a retry counter in the gate state; updated docs with escalation recommendation.
Noah Friedman: Request: add a deterministic seed option in the simulator to make CI reproducible. Also run the new e2e with the LRU cache disabled to validate edge cases.
Arielle Chen: Added --seed flag to simulator and added an e2e CI matrix row with cache off.
