# Reduce CI costs with test-slicing, sparse-checkout and dependency-minimizer

Motivation: CI costs and queue times have grown with the test matrix and the monorepo size. This PR introduces three coordinated changes to reduce CI runtime and cache pressure: (1) test-slicing: split integration/unit suites into deterministic shards by changed modules, (2) sparse-checkout in CI runners to limit workspace files downloaded for each job, and (3) dep-minimizer: a lightweight analysis tool that prunes unrelated dependency versions from the effective dependency set used for install caches.

Summary of changes: Added a test-slicing executor (tools/test-slicer) with heuristics for affected-package detection and deterministic shard assignment. Updated .github/workflows/ci.yml to run lightweight changed-file detection as a short-circuit step and only spin up full runners when needed. Integrated sparse-checkout logic into the checkout step to reduce workspace size for long tests. Added infra/dep-minimizer which analyzes lockfiles and module imports to produce a reduced lock overlay used only for CI, keeping production lockfiles untouched.

Why this is safe: All changes are opt-in and guarded by repository config flags (ci.test_slicing.enabled, ci.sparse_checkout.enabled, ci.dep_minimizer.enabled). The dep-minimizer outputs a diagnostics report and cannot be enabled unless a repo admin flips the flag. Test-slicing preserves global test order determinism by seeding shard assignment with repository-wide salt and PR number.

Testing and CI: Local unit tests for the test-slicer and dep-minimizer are included. The workflow contains a smoke job that runs a full-suite against a preserved baseline when the config is changed. In this PR the smoke job was run twice against main and head to validate parity. CI run summary appears in checks_summary.

Compatibility notes: No library API changes. Developers will notice faster pre-merge feedback and smaller checkout times in CI. Teams that rely on full workspace integration tests should opt into slicing gradually; docs/ci/onboarding.md includes migration steps and troubleshooting.

Checklist: - [x] Adds test-slicing implementation and unit tests - [x] Adds dep-minimizer implementation and unit tests - [x] Adds sparse-checkout integration into CI - [x] Feature flags and safe defaults - [x] Migration docs and rollout plan in docs/ci/onboarding.md

Rollout plan: 1) Merge opt-in feature flags and tools; 2) Enable per-team via repo config for low-risk services; 3) Monitor per-route latency and CI cache hit-rate for 2 weeks; 4) Gradually enable wider.

Notes for reviewers: Focus on correctness of affected-file detection and shard determinism (see tools/test-slicer/README.md). dep-minimizer intentionally avoids rewriting production lockfiles; verify the overlay generation and diagnostics in infra/dep-minimizer/README.md.
Evan Park (2026-02-13): Left a few nit comments on documentation and a question about shard seeding collision with external merges — can we base the salt on repo UUID instead of human-readable name?
Maya Thompson (2026-02-13): Reply: switched to repo UUID-based salt in next commit (8a1f3d2), added a short note in README explaining determinism guarantees.
Rita Alvarez (2026-02-15): Concern: dep-minimizer might miss dynamically imported modules in backend services. Suggest running fallback full-install when dynamic imports are detected.
Maya Thompson (2026-02-15): Reply: added heuristics to detect common dynamic import patterns and a conservative fallback that forces full install for affected packages (d4b7c9a).
Samir Patel (2026-02-16): Approved after verifying unit tests and running the smoke job locally. Left a comment to monitor cache hit-rate dashboards after rollout.
