# Split lint pipeline into micro-shards and add dependency-stability adapter

Motivation: Our monolithic 'lint-and-style' presubmit jobs have regressed in latency and flakiness as the repo grew; some sub-linters are expensive and unrelated to small changes, while transient dependency resolution failures increase reruns. This PR decomposes the lint job into small, language- and domain-aware micro-shards and introduces a lightweight Dependency Stability Adapter (DSA) that pins known-unstable transitive ranges during CI resolution.

What this PR does (single-string summary): - Introduces a lint-shard manifest and orchestrator that maps changed files to a minimal set of lint shards; - Adds three new composite GitHub Actions: lint-shard-runner, lint-shard-dispatch, dep-stability-adapter; - Implements a cache-aware pip/npm/yarn fetch step to reduce install time when shards run in parallel; - Adds a CI-time heuristic to skip expensive style-generation steps for documentation-only changes; - Adds telemetry hooks (metrics and spans) so Console can show per-shard latency and failure reason.

Why micro-shards: Large, language-agnostic jobs block the CI queue and make quick iteration painful. Micro-shards let us: parallelize across ephemeral runners, target only linters relevant to the diff, and retry only failing shards. We expect ~1.8x median feedback improvement on typical PRs and ~45% walltime reduction for broad code churn.

Dependency Stability Adapter (DSA): DSA is a non-invasive helper that runs before install step. It consults a small repo-maintained stability index (stability/index.json) and applies temporary .npmrc/pip constraints to pin flaky transitive ranges during CI resolution. This reduces transient network/version resolution failures and provides a signal for platform owners to escalate unstable upstream packages. DSA has a dry-run mode for rollout.

Compatibility & safety: - The changes only affect CI pipelines and GitHub Actions behaviors; no production runtime code changes. - DSA only modifies CI-local resolution (no lockfile commits) unless a maintainer explicitly runs the stabilization tool locally. - Feature is opt-in via repo-level workflow config; default rollout for the monorepo enabled after small pilot.

Checklist: - [x] Add lint-shard manifest generator (tools/ci/lint_shard_mapper.go) - [x] New composite actions in .github/actions/{lint-shard-runner,lint-shard-dispatch,dep-stability-adapter} - [x] Update top-level workflows to use staged lanes (fast-feedback, shard-parallel, full-check) - [x] Add telemetry events and metrics (console.integration tag) - [x] Add docs in docs/dev/ci.md explaining shard mapping & how to run locally - [x] Run pilot (5 teams) for 2 weeks and collect latency/failure metrics

Commits:
- feat(ci): add lint-shard manifest generator and mapping rules
- feat(ci): composite action lint-shard-dispatch to create per-shard jobs
- feat(ci): composite action lint-shard-runner with cache-aware installs and per-shard reporting
- feat(ci): dep-stability-adapter (CI-only pinning dry-run + opt-in mode)
- perf(ci): split existing workflow into staged lanes; add fast-feedback lane
- docs: document local shard execution and DSA behavior

Files changed (high level): .github/workflows/presubmit.yml (split lanes), .github/actions/lint-shard-* (new actions), tools/ci/lint_shard_mapper.go, stability/index.json (initial entries), docs/dev/ci.md, scripts/ci/cache-helpers.sh, package.json (ci scripts), internal/telemetry/ci_events.go (shard events). Smaller updates across various package roots to annotate language ownership for mapping.

Testing & CI: - Local unit tests for the shard mapper and DSA logic included. - E2E smoke run on a larger feature branch (redwood/monorepo sample) exercised shard dispatch with 8 concurrent runners. - CI initially hit a flaky network resolution on second run; after enabling DSA in dry-run -> full mode, resolution failures dropped by 87% in the pilot.

Rollout plan: 1) Merge in main with DSA defaulted to dry-run; 2) Run pilot on 5 active teams for 2 weeks; 3) Switch DSA to enforcement and monitor nightly stability report; 4) Gradually expand fast-feedback lane to all PRs and tune shard mapping rules.

Monitoring & rollback: - Added Console dashboards for shard latency and per-shard failure rate. - If shard failures exceed threshold or new flakiness appears, we can toggle to fallback monolithic lane via feature flag in .github/workflows/presubmit.yml.

Related work: ENG-2301 (CI feedback latency reduction), ENG-2387 (transient dependency noise), PRs: internal refactors to split presubmit runner in prior cycles.

Notes for reviewers: Focus review on - the correctness of file-to-shard mapping rules (tools/ci/lint_shard_mapper.go), - potential gaps where a shard may miss relevant linters, - DSA index entries and opt-in gating, - caching semantics in composite actions (cache key patterns).

Liam Ortega: Please add a guard so that docs-only PRs don't trigger any non-doc linters; also verify cache key entropy to avoid stampede.

Evelyn Park (author): Added docs-only heuristic and tuned cache key to include repo path + toolchain hash; added tests for docs-only mapping.

Priya Nair: The shard mapper rules look good but add a fallback that runs full lint on PRs that touch >20 top-level groups.

Evelyn Park: Added fallback threshold and documented the rationale.

Marco Ruiz: Approved after verifying telemetry events and CI action timeouts.

Continuous Integration Bot: All required checks completed successfully.

Reduce premerge CI walltime by ~45% for large PRs via lint micro-shards; add dependency-stability adapter to detect and pin flaky transitive updates; no runtime behavior changes
feat(ci): add lint-shard manifest generator and mapping rules
feat(ci): composite action lint-shard-dispatch to create per-shard jobs
feat(ci): composite action lint-shard-runner with cache-aware installs and per-shard reporting
feat(ci): dep-stability-adapter (CI-only pinning dry-run + opt-in mode)
perf(ci): split existing workflow into staged lanes; add fast-feedback lane
docs: document local shard execution and DSA behavior
