# Introduce CI health prober + sparse dependency curator; slim presubmit dark-launch

Motivation: CI feedback latency and spurious presubmit failures have been increasing as the repo grows. Two compounding causes: (1) flaky external checks masking unhealthy runners, and (2) a large surface area of transitive devdeps where stale packages cause occasional lint/test regressions. This PR introduces a lightweight CI health prober alongside a sparse dependency curator and a dark-launchable presubmit path to reduce noise while enabling incremental rollout.

What this change does (summary):
- Adds a small prober service (.github/workflows/ci-health-prober.yml + tools/ci/prober) that periodically validates runner images, network egress to common package registries (npm/pypi/go proxy), and the workspace mount behavior that has been the root of several flakes. The prober exposes a simple JSON health endpoint consumed by the CI dispatcher to deprioritize flaky runners.
- Introduces a sparse-dep-curator CLI (tools/dep/curator.py) that computes a minimal devdep set based on changed packages and a light heuristic to prefer pinned semver ranges. The curator emits a delta manifest consumed by presubmit bootstrappers to avoid installing the entire monorepo devdep graph on short-lived CI lanes.
- Adds a dark-launchable presubmit shim (scripts/presubmit_darklaunch.py) which can be enabled per-team via repo config. When enabled in dark mode the shim runs the new fast-path (curator + small bootstrap) alongside the full presubmit and reports parity metrics without blocking merges. The idea is to gather confidence and telemetry before flipping the fast path for all repos.
- Small refactors to existing lint/test bootstrappers to accept a curated manifest and fall back to full install when the curator is unavailable.

Why this is safe: changes are opt-in by default (dark-launch off), prober runs only read-only checks and timeouts are controlled, and the curator never mutates lockfiles or registry state. Rollout plan is included in the checklist.

Testing performed:
- Local unit tests for curator heuristics (tools/dep/tests).
- Integration run in ephemeral staging CI pool for 48h comparing: full-presubmit vs curated-presubmit; parity rate for lint/pytest was 99.1% over 3200 PRs sampled in staging.
- Prober deployed to staging runners for 72h; dispatcher successfully suppressed two runners exhibiting stale network mounts.
- Manual QA: enabling dark-launch for one team for 72h; metrics collected via /internal/ci-health (see rollout notes).

Checklist (complete):
- [x] Add prober workflow and service code
- [x] Add sparse-dep-curator CLI + tests
- [x] Make presubmit bootstrappers accept curated manifest
- [x] Add dark-launch toggle in repo config and runtime flag
- [x] Add minimal observability (metrics + parity logging)
- [x] Documentation: docs/ci/health-probe.md and docs/ci/curator.md
- [x] Integration testing in staging pool

Rollout notes: start with dark-launch for one team (config: teams/infra/ci-config.yaml), run for 2 weeks, validate parity metrics and error budget. If parity > 99% and prober reduces runner flakiness, flip to default for short-lived lanes and keep full install for long tests for three more weeks before full migration.

Related work: ENG-6215 (ci health observability), ENG-6217 (dep surface reduction).

Notes for reviewers: the core logic for the curator is intentionally conservative — it prefers correctness and reproducibility over aggressive trimming (we only remove packages if they're unused by changed packages or devtools with explicit allowlist). The presubmit shim includes a telemetry flag to trace false positives (curator trimmed something that led to a real break).

Ethan Moore: Could we add a timeout guard around the network checks? Prober should never block runner lifecycle. -- Done in 18a3f9d; timeouts 10s.
Lina Kovács: Add a flag to force full install for flaky PRs while we evaluate parity. -- added --force-full in presubmit shim.
Omar Hassan: Curious if curator handles private internal packages behind our artifactory proxy. -- curator supports registry mapping via CURATOR_REGISTRY_MAP; default map added to team config.
CI run summary: initial pipeline failed on lint due to shellcheck in presubmit bootstrap; fixed in follow-up commit. Final run: all checks passed (unit, integration smoke, e2e staging).
Introduce CI health prober to detect flaky runners and a sparse dependency curator to speed up presubmit installs. Dark-launchable presubmit path added to gather parity before full rollout. Opt-in; no breaking changes.
- Turn on dark-launch for infra team (teams/infra/ci-config.yaml)
- Run parity rollover analysis after 2 weeks
- Add curator metrics to CI dashboard (observability team)

Prober runs use runner credentials to probe network only; add CURATOR_REGISTRY_MAP to enterprise installs that use private registries. See docs/ci/health-probe.md for incident runbook.
