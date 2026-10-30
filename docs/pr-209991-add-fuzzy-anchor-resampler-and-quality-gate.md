# Add fuzzy anchor resampler and quality gate

Adds a probabilistic resampling-based stability scorer and CI quality gate that flags semantic sensitivity regressions for prompt cohorts. Configurable thresholds and local CLI for repros.
Motivation: Prompt stability regressions can be subtle — small semantic or lexical perturbations of anchor prompts sometimes reveal brittle model behavior that deterministic CI checks miss. This PR introduces a lightweight fuzzy anchor resampler and a semantically-weighted stability score that plugs into the eval-harness CI pipeline as a soft/hard quality gate.

What this changes: Adds a resampling simulator that generates small, semantics-preserving perturbations of anchor prompts using paraphrase templates and embedding-distance constraints; runs the model under test on the sampled variants; computes token-level divergence, semantic drift (embedding delta), and an aggregate stability metric that weights cohorts by production exposure and semantic density. The CI hook consumes the aggregate score and can be configured to either soft-fail (comment and block merge approval) or hard-fail (fail CI) based on project-level thresholds.

Design highlights: - Fuzzy resampler: Monte Carlo sampler with pluggable perturbation strategies (synonym substitution, sentence-split/recombine, controlled paraphraser). Sampling is constrained by an embedding similarity window to avoid unrealistic paraphrases. - Semantically-weighted stability score: Combines normalized token divergence, BLEU-like alignment, and embedding delta into a 0-100 stability score; cohorts with higher production traffic receive proportional weight. - CI adapter: ci/hooks/fuzzy_anchor_gate.py integrates with existing eval CI admittance layer. It emits structured diagnostics (top failing prompts, exemplar variants, per-cohort breakdown) as a single-line JSON blob in the CI logs for downstream parsers. - Local tooling: a small CLI (eval-harness/tools/resample_cli.py) to run the same evaluator locally for debugging and reproducing gated failures.

Backward compatibility & rollout: Feature is behind the feature flag "fuzzy_anchor_gate" in eval-config. Default behavior is disabled for existing pipelines; repo owners can opt-in per-Linear project or per-branch. A staged rollout plan was used: local dev testing, canary on a single repo, then org-wide opt-in.

Testing: Unit tests for resampler strategies and score normalization were added (tests/test_resampler.py, tests/test_score_normalization.py). Integration smoke tests simulate a toy prompt cohort and assert that the gate emits expected diagnostics. CI run attached to this PR passed after two reruns (flake in a network call to the paraphrase service was retried).

Usage: Example config in docs/tests/qa-guide.md shows how to enable the gate with a soft-fail threshold (stability &lt; 72) and an optional hard-fail threshold (stability &lt; 60). The CLI supports a --repro flag to save failing variants to a local directory.

Notes on model cost: Sampling defaults to 12 variants per anchor with an adaptive early-stop if divergence converges; this keeps additional cost bounded in CI. Teams can lower the sample count for more frequent runs.

Related work: complements existing anchor and replay mechanisms (see ENG-8421) by focusing on semantic robustness rather than deterministic regression.

Questions welcome — I included an example run output in the repo docs and linked the Linear tasks that drove this work.
Jordan Patel: Nice direction — can you add a brief note on compute budget controls?
Samira Lopez: Added adaptive early-stop and sample_count override; documented in the CLI and config (see feature_flags.yaml).
Miguel Santos: Did you consider using anchor salience weights coming from traffic telemetry?
Samira Lopez: Yes — the scorer accepts an optional exposure_weight map; default is uniform. We'll wire telemetry in a follow-up (ENG-8476).
Avery Kim: Approve after addressing logging verbosity and adding a compact CI artifact for quick triage.
Samira Lopez: Reduced logging level and added a concise CI artifact JSON with top-5 failing prompts.
