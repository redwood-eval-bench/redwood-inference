# Unify checker-profiles, add auto-deprecation-mapper, and devcontainer-aware presubmit lane

Motivation: Our presubmit lanes have accumulated bespoke lint/test configuration fragments across services and SDKs (node/python/go). This PR consolidates those into a small set of extensible 'checker-profiles' that encode sets of linters, shims and environment knobs, and additionally introduces an automated deprecation mapper that surfaces and labels transitive dependencies flagged for deprecation during CI. Finally, the presubmit runner now supports a devcontainer-aware quickpath so contributors on devcontainers get a fast local-equivalent feedback lane.

What this changes (single-string summary): Consolidate per-repo lint/test configs into profiles; add a binary 'deprecation-mapper' that parses lockfiles and advisory feeds to emit a deprecation score per dependency; wire deprecation metadata into PR annotations and the presubmit dashboard; introduce a devcontainer-aware presubmit lane that uses lightweight warmed caches and skips heavy integration shards when run locally; refactor existing composite actions to be profile-driven; add telemetry hooks (counter gauges) to measure presubmit latency and dep-staleness over time.

Checklist: - [x] Define stable checker-profile schema and backward-compatible shim layer - [x] Implement auto-deprecation-mapper and adapters for yarn/pnpm/pip/poetry - [x] Integrate mapper output into PR annotations and console UI - [x] Add devcontainer quickpath and cache-warm hook - [x] Update CI templates to accept profile param and enable opt-in quickpath - [x] Add unit + integration tests for new actions - [x] Run staging CI across representative monorepo subsets

Commits (high-level): - initial: introduce checker-profile schema, docs and sample profiles - action: refactor composite actions to accept profile input and env overrides - tool: add cmd/deprecation-mapper with parsing and advisory adapters - infra: devcontainer quickpath runner + cache-warm script - tests: add unit tests for mapper and e2e smoke for profile routing - docs: presubmit migration guide and migration script

Files changed (high level): - .github/actions/checker-profiles/* (new) - .github/workflows/presubmit.yml (updated to route by profile) - tools/deprecation-mapper/* (new CLI + adapters) - infra/devcontainer/quickpath-runner.sh (new) - docs/ci/checker-profiles.md (new migration guide) - sdk/python/.lint.yml, sdk/ts/.lint.yml, services/*/.presubmit (removed duplicates) - tests/ci/e2e/presubmit-profiles (new smoke harness)

Impact and risk: The change reduces duplication and should lower CI latency for local runs. Risk areas: the mapper may initially surface many deprecation warnings (noise) and some services rely on bespoke linter flags removed by the profiles. To mitigate we ship a compatibility shim and an opt-in flag to keep legacy behavior until teams migrate.

Rollout plan: 1) Merge to main with feature flag off for global presubmit. 2) Enable profile-driven presubmit for a small set of repos (sdk/python, redwood-docs) and collect telemetry for two weeks. 3) Gradually flip on default profile on a per-team basis, using the compatibility shim where needed. 4) After 4 weeks of telemetry with low noise, enable PR annotations for deprecation warnings by default.

Release notes: Consolidates per-repo lint/test config into shared 'checker-profiles', introduces an automated dependency deprecation mapper with PR annotations, and adds a devcontainer-aware quickpath to speed contributor feedback. Teams should read docs/ci/checker-profiles.md for migration steps.
Asha: Can we make the mapper rate-limit advisory feeds?
Author (Lina): Yes — mapper caches feeds for 24h and falls back to local lockfile heuristics when feeds are unavailable; added tests for cache expiry.
Diego: The profile schema looks good but a couple repos have custom eslint rules — how will migration work?
Author: We added a compatibility shim that reads legacy .eslintrc and translates to the profile; migration guide includes examples.
Samir: CI spike observed on first staging run (more CPU on presubmit runners).
Author: Added cache-warm and opt-in quickpath so local runs avoid heavy shards; reduced default concurrency for staging until we evaluate telemetry.
Asha (approve): Approving after squash commit hygiene; please include release notes.
Diego (comment): Suggestion to add a small telemetry event for 'mapper.noise_count' (done).
Merge: Squash merged by Lina via web UI after CI green on 2026-02-18.
Staging: flaky at first (one lint shard timeout) -> rerun -> all green. Main CI: all checks passed (lint/unit/e2e smoke). Coverage: no significant regression. Deployer smoke: OK.
Shared checker-profiles for presubmit; automated dependency deprecation mapper that annotates PRs; devcontainer-aware presubmit quickpath; compatibility shims for legacy configs.
Focus on the checker-profile schema (docs/ci/checker-profiles.md) and the compatibility shim for tslint/eslintrc conversions. Verify the mapper adapters cover our top-10 package managers. Check telemetry naming conventions.
1) Monitor telemetry dashboard (ci.presubmit.latency, deps.deprecation_score) for 14 days. 2) Reach out to teams with >=10 mapper warnings in a week for guidance. 3) Iterate on profiles based on feedback.
