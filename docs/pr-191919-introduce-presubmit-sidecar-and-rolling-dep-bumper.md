# Introduce presubmit sidecar runner and rolling dependency bumper workflow

Context: Our presubmit lane has grown monolithic: lint, test orchestration, and dependency hygiene are all handled by a single runner which results in slow startup, noisy caching, and frequent interruptions when we update tooling. This PR decomposes the presubmit orchestration into a small language-aware sidecar runner and introduces a scheduled rolling dependency bumper workflow to keep devdeps fresh without spamming PRs. Motivation: Reduce presubmit latency for fast feedback, isolate language tooling concerns, and automate safe minor dependency bumps so teams can focus on feature work. This also primes the repo for future lightweight local CI images.

What I changed (high level):
- Added .github/workflows/presubmit-sidecar.yml: new workflow that launches a persistent presubmit sidecar container for language-aware runners to connect to; optimizes warm caches and shares toolchain leases.
- Added tooling/presubmit/sidecar/README.md and sidecar runner service (Go): sidecar implements a tiny RPC for job registration, incremental test selection, and lint job routing by language.
- Split existing monolithic presubmit entrypoints into language plugins under tooling/presubmit/plugins/{go,js,py}. Each plugin is lightweight and focuses on the language-specific linter/test invocation.
- Introduced .github/workflows/dependency-bump-rolling.yml: weekly scheduled workflow that runs dependency intelligence, creates batched bump PRs scoped to teams, and tags owners for review.
- Added scripts/dep-bumper/main.py: the core bump logic with opt-in rules, safety checks, and changelog aggregation.
- Updated docs/ci/presubmit-migration.md with rollout/migration instructions and how to opt out teams during the ramp.
- Small lint/config changes: .golangci.yml tweaks and package.json devdep pruning.

Why this approach: The sidecar keeps heavyweight toolchain boot time amortized across multiple presubmit runs and avoids rebuilding caches for each ephemeral job. Language plugins allow teams to own and fast follow linter/test updates without touching the orchestration layer. The rolling dependency bumper reduces PR noise by batching minors and auto-annotating risk level. Compatibility: Sidecar exposes a minimal contract; legacy presubmit entrypoints are preserved for three weeks and flagged by a deprecation linter that emits a non-failing diagnostic.

Testing and CI:
- Local integration tests added for the sidecar RPC protocol (tooling/presubmit/sidecar/sidecar_test.go).
- Unit tests for the dep bumper with fixture lockfiles (scripts/dep-bumper/tests/*).
- CI pipeline: initial runs showed intermittent flakes on go-test; fixed by pinning the test cache path in runner. Full CI: lint / unit / integration / smoke passed on final run. See ci checks summary below.

Migration and rollout: 1) Merge this PR to main with feature gate disabled by default. 2) Enable sidecar on an opt-in message in team repos (via a small config file .presubmit.yaml). 3) After two weeks of observation we will flip default to on and deprecate legacy entrypoints in the linter. 4) Rolling dependency bumper starts in "dry-run" mode for one month before opening PRs.

Checklist:
- [x] Sidecar runner implemented and unit tested
- [x] Language plugins moved and smoke-tested
- [x] Dep bump script + scheduled workflow added
- [x] Docs and migration guide included
- [x] Backwards compatibility for three weeks
- [x] Reviewed and approved by infra and SRE

Notes and follow-ups:
- Follow-up ticket ENG-1420: add metrics for sidecar cache hit rate and presubmit latency per-language.
- Follow-up ticket ENG-1421: add circuit-breaker if sidecar becomes overloaded.
- This change touches the presubmit surface; expect a small increase in e2e dev infra usage during the ramp.
Evan Chu: Nice decomposition. Two asks: 1) can the sidecar accept immutable leases for cross-PR caches so we don't leak artifacts? 2) add a health endpoint for SLO alerting.
Author (Maya): answered — added lease scoping and a /health endpoint with simple prometheus metrics; clarified in README.
Lena Martinez: dep-bumper looks promising. Request: make batch size configurable per team and add a changelog snippet for reviewers.
Author (Maya): addressed — batch size is config-driven and bumper now aggregates changelog fragments into the PR body.
Tomás Rivera: small nit on plugin scripts — prefer bash strict mode and fewer subshells.
Author (Maya): updated entrypoint.sh for all three plugins with set -euo pipefail and comments.
SRE (automated comment): observed increased presubmit container count during integration tests; recommend rolling enablement.
Author (Maya): noted in migration guide and added default-off gating.
Merged to main via squash after approvals from infra and SRE. All CI checks green on final run. Migration gated by feature flag and documented.
- Introduce presubmit sidecar to reduce presubmit latency and centralize toolchain leases.
- Add language-specific presubmit plugins (Go/JS/Py).
- Add scheduled rolling dependency bumper (dry-run initially).
Keep legacy presubmit entrypoints available for at least three weeks. Monitor sidecar cache hit rate and test runner queue length (ENG-1420). If load spikes, flip the feature gate off and open follow-up to tune leases.
