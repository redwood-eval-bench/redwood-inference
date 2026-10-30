# Introduce context-aware lint adaptor and quiet-presubmit mode to reduce premerge noise

Motivation: our monolithic presubmit lint runner is noisy and slow for multi-language changes. Small edits (docs, infra) often trigger heavy linters and CI flakes which slow down iteration. This PR introduces a lightweight, context-aware lint adaptor that maps changed files to a minimal set of language-specific lint checks and a new quiet-presubmit mode that downgrades non-actionable warnings into summary notes unless a threshold of failures is reached.

What this PR does: introduces a lint adaptor service (tools/presubmit/lint-adaptor) that inspects the file change set and emits a targeted lint profile; replaces the single heavy lint job in .github/workflows/presubmit.yml with staged lanes (quick, language-specific, full) and a quiet-presubmit flag that maintainers can toggle per-repo. Centralizes presubmit config into ci/presubmit-manifest.yaml to avoid ad-hoc scripts across repos. Bumps ESLint and golangci-lint minor versions and pins a couple of GitHub Action SHAs for stability. Adds unit tests for the adaptor mapping and a smoke test for the quiet-presubmit flow.

Rollout plan: staged opt-in on a subset of small repos for 48h, monitor premerge pass rate and tokenized latency; then enable org-wide with a one-week monitoring window. If rate of missed actionable failures > 0.1% we will revert quiet-presubmit by repo until fixed.

Checklist: [x] lint adaptor implementation; [x] presubmit workflow update; [x] unit and smoke tests; [x] docs for maintainers (ci/README-presubmit.md); [x] pinned CI actions for deterministic runs; [x] Linear tickets updated.

Notes: quiet-presubmit defaults to disabled for redwood-private-installer and other security-critical repos. The adaptor is intentionally conservative: it never disables typecheck/lint jobs for files in core/runtime/ or infra/ folders.
Marcus Chen: Nice approach — can the adaptor misclassify a file and skip a critical check? How do we fail-safe?
Eleanor Park: The adaptor is conservative by design. Any change touching runtime/, infra/, or security/ will route to the full lane. We also have a fallback that runs full lint on merge if quiet-presubmit was active to avoid silent regressions.
Aisha Patel: CI flapped on first run due to an action version mismatch, saw failures in the smoke job. Re-run fixed it — consider pinning more shas?
Eleanor Park: Added .github/actions/pin-versions.txt and pinned the 2 action steps that previously caused flakiness. Left a note in ci/README-presubmit.md for future updates.
Diego Rivera (request changes): Please add a short perf guard so quick lane never runs more than 3 linters in parallel to avoid token spikes on small repos.
Eleanor Park: Added concurrency cap in presubmit-manifest.yaml and updated docs. Re-ran tests; all green.
Marcus Chen: Approved after the concurrency cap and smoke tests. Looks safe to merge.
To opt a repo into quiet-presubmit set ci/presubmit-manifest.yaml::repos.<repo>.quiet_presubmit = true and monitor the quiet_presubmit/alert metric. The adaptor mapping lives in tools/presubmit/lint-adaptor/mapping.yaml — add language groups there for new languages. See ci/README-presubmit.md for troubleshooting.
Introduce a context-aware lint adaptor and quiet-presubmit mode to reduce noisy premerge failures and speed up developer feedback. No user-facing API changes.
