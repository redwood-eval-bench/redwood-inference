# add cross-domain sensitivity curator CI hook

Motivation: Flaky regressions often come from prompt-sets that behave inconsistently across domains (e.g., release notes vs. casual chat). This PR introduces a CI hook that computes a cross-domain sensitivity profile per prompt and surfaces high-sensitivity prompts to the regression gate so teams can decide on targeted fixes or soft-fail exemptions.

What this adds: a lightweight 'sensitivity curator' module that: (1) samples model outputs across a configurable set of domain contexts, (2) computes a normalized sensitivity score using existing eval-harness metrics (semantic divergence, token-level entropy, and answer-stability overlap), (3) annotates prompt metadata with cohort-level sensitivity tags, and (4) integrates with the CI gate to mark a prompt-set as "sensitivity-noisy" if the median sensitivity exceeds a threshold.

Behavior in CI: by default the hook is a soft-fail (reports failures but does not block) for new prompt-sets; once a prompt-set is onboarded and manually approved, teams can flip the hook to hard-fail to prevent high-sensitivity regressions from merging. The hook includes a per-PR summary that lists top-10 most sensitive prompts and a small example matrix of responses across domains to aid triage.

Implementation notes: the module reuses the existing eval-metrics adapter and the cohort sampler; sensitivity is computed as a weighted combination of semantic cosine deviation (embedding-based), answer-structure divergence (BLEU/ROUGE style similarity for structured outputs), and response entropy variance across domains. The config is exposed under eval-harness/config/sensitivity_curator.yaml with reasonable defaults and a short doc section describing the thresholds and onboarding flow.

Backwards compatibility: default thresholds preserve existing gate behavior (soft-fail only). The module is feature-flagged and can be toggled off via ENV or CI variables.

Testing and validation: added unit tests for score normalization and an integration smoke test that runs against the local sample prompt-sets in testdata/cross_domain_samples.json. Benchmarks show the hook adds ~4s median latency to the full eval pipeline for a 200-prompt set; we made the sampling strategy configurable to avoid CI slowdowns for large sets.

Related work: this complements existing prompt-cohort SLO checks and the sensitivity-targeted replay hooks; it is intended to reduce noisy CI failures and improve the signal-to-noise ratio of regression alerts.

Acceptance criteria: merged once reviewers approve and CI green; rollout gated behind a feature flag and documented in the README section for CI hooks.
Liam O'Rourke: Overall nice design. A few nits: ensure the default threshold is conservative and add telemetry counters for how often prompts are flagged.
Maya Chen: Added telemetry counters (prompts_flagged_total, sensitivity_score_histogram) and bumped default to conservative value. Updated CI summary format to include counters.
Priya Nambiar: Could we reduce memory for large prompt-sets? The integration test OOMed locally with 5k prompts unless sampling is enabled.
Maya Chen: Implemented reservoir sampling path and a sample-size cap. Documented the sampling config and added a smoke test for the reservoir sampler.
Diego Alvarez: Tests look good. Suggest we expose an opt-in strict mode for teams that want immediate hard-fail behavior.
Maya Chen: Added feature-flag support and documented strict_mode env var in config file.
Adds a cross-domain sensitivity curator CI hook that detects prompt-sets with high variability across domains and surfaces them in CI summaries. Default is soft-fail; opt-in strict mode available.
