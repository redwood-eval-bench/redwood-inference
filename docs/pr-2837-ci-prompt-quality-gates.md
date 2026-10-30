# add ci quality gates for prompt-regression checks and eval-metrics integration

Motivation: We need automatic, fast feedback when prompt-set regressions or metric drifts occur. This PR adds a lightweight CI quality gate that runs a subset of the evaluation harness on PRs touching prompts, eval configs, or model-adapters. Gate decisions are based on configurable metric thresholds and on per-prompt regression checks.

What this does (single continuous description): Adds a new GitHub Actions workflow (/.github/workflows/eval-gate.yml) that runs a staged evaluation: 1) quick-run of the canonical regression prompt-sets with a strict timeout and sampled inputs; 2) compute core eval metrics (exact-match, F1, BLEU-lite, response-latency median) via the new eval-harness/metrics/adapter so integrations are consistent with downstream dashboards; 3) apply quality gate checks defined in eval-harness/quality_gates.yaml which include per-prompt allowable delta, global metric floors, and an opt-in strict mode for sensitive repositories; 4) if the gate fails, the workflow posts an annotated comment with a short summary and a link to the full artifacts; 5) if the gate passes, the PR can proceed and a green check is emitted.

Additional features: - New CLI flag eval-run --ci-sample to run a fast sampled evaluation in CI. - A light-weight cache for prompt outputs to speed reruns (used only in the workflow runner). - A documented fallback: maintainers can re-run the full eval job manually from the console with --ci-sample=false.

Why this is safe: The CI step defaults to a small sample size and strict wall-clock timeouts to keep run-time under the existing CI budget. The added files are isolated to the eval-harness repository and do not alter runtime model-serving code.

Rollout plan: - Initially gate only PRs that change prompt-sets, eval configs, or the eval-harness code paths (filtering is implemented in the workflow). - After two weeks of monitoring and tuning thresholds (ENG-4123), enable gates for SDK docs and examples.

Related: ENG-4123, ENG-4271. See the new docs at eval-harness/docs/quality-gate.md for operator guidance.
Ethan Ruiz: Can we make the sample deterministic across runs? Flaky CI will erode trust.
Maya Patel: Good call — added a deterministic seed derived from PR number when --ci-sample is used.
Priya Menon: Thresholds look aggressive for a first pass; recommend widening tolerances for BLEU-lite by +0.05.
Maya Patel: Relaxed BLEU-lite thresholds and added a 'strict' opt-in field in quality_gates.yaml so teams can opt into tighter checks.
Omar Haddad: CI artifacts are large — please upload only the JSON summary and the failing example set.
Maya Patel: Adjusted artifact upload to include summary.json, failing_examples.json, and a compressed tarball of raw outputs (max 10MB).
