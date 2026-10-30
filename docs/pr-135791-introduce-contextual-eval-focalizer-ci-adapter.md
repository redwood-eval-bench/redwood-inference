# lane-aware contextual eval focalizer: CI adapter for intent-segmented prompt checks

Motivation: some regressions only surface for particular user-intent cohorts or usage lanes (short queries vs long-form, transactional vs exploratory). This PR adds a CI adapter that selects and weights prompt-set slices (lanes) by contextual metadata, runs lane-specific evals in parallel, and synthesizes a lane-aware regression signal that gates merges only when meaningful lanes degrade.

What this change implements: contextual_focalizer CI adapter + lane aggregator + lightweight lane-profile config. The adapter: 1) reads lane definitions from eval-config/lane-profiles.yaml (static but overridable by CI env), 2) samples prompts by lane using feature selectors (intent tag, avg_tokens, user_tier), 3) runs eval shards in parallel (respects existing eval runner), 4) aggregates lane-level metrics (accuracy, semantic-similarity, token-efficiency) into a weighted lane-score and exposes a compact JSON report for the CI pipeline.

Why this approach: avoids single-number masking of regressions across heterogeneous usage; enables targeted rollbacks or canaries for affected lanes; integrates with existing smoke/eval gates and is designed to be low-cost (sample budgets per lane).

Checklist: 
- [x] Add adapter implementation and unit tests
- [x] Add integration test pipeline config (ci/workflows/lane-eval.yml)
- [x] Add lane-profiles.yaml with examples and docs
- [x] Ensure deterministic seeding for sampling to avoid flakiness
- [x] Add metrics exporter hook for Redwood Console ingest

Compatibility notes: The adapter is opt-in via CI env VAR EVAL_LANE_FOCALIZER=1. Default lane profiles are conservative; existing eval runners unchanged unless flagged.

Related Linear tasks: ENG-2471 (lane-aware eval prototype), ENG-2505 (CI gating UX follow-up).
Miguel Rivera: Nice direction — can we avoid sampling bias when lane sizes are imbalanced? Suggest adding per-lane min-sample floor and exposing effective sample count in the report.
Asha Patel: Added per-lane min_sample in lane-profiles.yaml and included effective_sample_count in the JSON report (see tests).
Claire Zhou: Please confirm the seeding guarantees across parallel runners; tests flaked locally for me. Also ask to make the default thresholds configurable (not hard-coded).
Asha Patel: Deterministic seed fixture added. Thresholds moved into eval-config/thresholds and CI reads them from EVAL_THRESHOLDS path; updated docs.
Samir Gupta: Quick nit — exporter hook should tag metrics with lane name and sample_count. Also we should rate-limit export to reduce cardinality. Could you add a small counter?
Asha Patel: Exporter now emits lane tags and a summary metric (lane_sample_count) and rate-limits to one export per minute per lane when run in parallel; implementation uses an in-memory debounce during a run (CI lifespan).
