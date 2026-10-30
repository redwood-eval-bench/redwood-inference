# Introduce CI step fencing and dependency priming to reduce noisy reruns

Context: Our CI has had noisy long-running reruns where expensive integration or e2e steps re-execute even for docs/infra-only changes. This inflates queue times and developer feedback latency. Motivation: cut wasted GPU/runner time and make presubmit faster while preserving safety of release-bar checks.

What this PR does: Adds a 'ci-step-fence' mechanism and a small 'dep-prime' job. Step fencing: we tag expensive jobs (integration, heavy-lint, full-e2e) with a conditional fence that only allows execution when a change touches codepaths listed by lightweight detectors. Dep priming: introduce a tiny early job that fetches and validates dependency metadata (lockfile fingerprint, quick checksum) and caches resolution artifacts to avoid cold-start flakiness during heavy steps.

Key behaviors: 1) Presubmit fastlane will run quick checks (typed-lint, unit) immediately. 2) Dep-prime runs in parallel and primes remote caches. 3) Expensive jobs check fence signals — if fence conditions are false we skip the heavy job and mark as skipped with a clear annotation. 4) Release and merge gates still run full checks on protected branches (we do not skip merge blocking release pipelines).

Migration notes: The fence detector lives in ci/detectors and can be extended per-language. Default detectors include Python imports, Go module changes, dockerfile/runtime changes, and changes to infra manifests. Operators can add repo-level opt-outs under .ci/fence-config.yaml.

Checklist: - [x] Add fence gating logic to composite actions and CircleCI workflows - [x] Implement dep-prime job and cache hints - [x] Add telemetry counters and a new lightweight CI metric (ci.fence.skipped_count) - [x] Backfill docs in docs/ci/fast-feedback.md - [x] Run a 48h experimental rollout on canary org - [x] Add a migration note for on-call and SRE

Testing & Rollout: Ran canary on 10 repos for 48h (observed 18% reduction in runner-seconds and 22% median presubmit latency improvement for non-code changes). Manual verification for merge-gate behavior and protected branch invariants. CI was run 4 times during review due to a flaky cache key; final run passed all checks.

Risks & Mitigations: There's a risk of incorrectly skipping a heavy job if detector rules are too permissive. Mitigation: default is conservative; add explicit file globs to force-run heavy jobs via .ci/fence-config.yaml and provide an admin bypass label [ci/run-heavy]. Also added alerting for unexpected skip ratios in the first 2 weeks.
Reduce CI flakiness by fencing expensive steps and adding a lightweight dependency priming step. No user-facing changes. Improves feedback latency for presubmit runs.
Ethan: Left a few comments on the detector heuristics, asked for one extra unit test around dockerfile change detection.
Priya: Asked for clarity around protected branch behavior; requested explicit test asserting merge gate never skips.
Diego: Approved after CI green and a short load run on canary.
Author replies: Addressed dockerfile detector test and added an explicit protected-branch check in presubmit workflow. Clarified docs on opt-out and added an escape hatch label.
Final status: approvals from Ethan and Priya; Diego provided an explicit +1 on canary metrics.
Presubmit run 1: failed (flaky remote cache key in dep-prime); rerun job after cache key tweak.
Presubmit run 2: pass (fast lanes and dep-prime passed; heavy jobs skipped as expected on docs-only changes).
Canary load test: 48h live; observed reduced queue depth.
Coverage: unit tests added for detector logic (approx 78% coverage on the ci/detectors package).
Merged via squash on 2026-02-21 by Maya Rao after required approvals and successful canary. Merge method: squash (single commit).
If CI shows unexpectedly high skip rates (>50% of heavy jobs skipped in a single day) alarm will fire to #sre-ci. To force-run heavy jobs for a PR add label 'ci/run-heavy' or modify .ci/fence-config.yaml.
