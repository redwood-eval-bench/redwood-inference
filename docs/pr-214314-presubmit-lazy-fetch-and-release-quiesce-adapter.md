# Introduce presubmit lazy-fetch + release quiesce adapter to reduce CI noise and fetch IO

Motivation: Presubmit pipelines have grown noisy and network-bound as more teams vendor large toolchains (language servers, preload artifacts, cached wheels). This PR implements a 'lazy fetch' presubmit strategy that defers heavy toolchain downloads until the presubmit job actually needs them, and a lightweight release 'quiesce adapter' that prevents release automation from triggering too aggressively during high presubmit churn.

What changed: consolidated a small presubmit shim and a release adapter: 1) tools/presubmit/lazy_fetcher.go - core logic: checks a lightweight manifest and only fetches big blobs when a stage declares the need; supports local cache stamp and HTTP range accepts. 2) .github/workflows/presubmit.yml - wire up a new lightweight presubmit-fastpath step and conditionally skip heavy setup steps. 3) cmd/release/quiesce_adapter/main.go - HTTP adapter used by release pipeline to delay merge windows when presubmit-load is high. 4) docs/ci/presubmit-lazy-fetch.md - runbook and configuration knobs. 5) charts/ci/values.yaml - expose quiesce thresholds and metrics config. 6) Makefile - convenience targets.

Design notes: the lazy_fetcher uses a tiny JSON manifest (kept in repo metadata) that maps steps to artifact classes (small/medium/large). CI runners consult the manifest; if step is marked large, the runner checks a shared cache and only downloads on miss. The release quiesce adapter polls presubmit-metrics (via existing CI telemetry endpoint) and will return an HTTP 202 to the release controller to request a 5-minute backoff when rolling error or queue depth thresholds are exceeded. The adapter is intentionally minimal and runs as a short-lived container in the release job.

Testing: unit tests for lazy_fetcher and quiesce_adapter included; added an e2e smoke job in .github/workflows/presubmit-smoke.yml that simulates a cache-miss heavy job and verifies fallbacks. CI matrix includes linux/go1.19 and go1.20 targets. Local dev instructions added to docs.

Rollout plan: feature-flag gated (env PRE_SUBMIT_LAZY=on) and staged rollout: enable in ci-fastlane for 1 week, then enable in full presubmit after monitoring. Quiesce adapter will ship with a conservative threshold and be toggled by release team during a 2-week observation window.

Why this is safe: No user-facing API changes. Presubmit steps will only be skipped when safe (manifest indicates optional heavy steps). The quiesce adapter only delays release merges by short windows and has a bypass parameter for critical security releases.

Follow-ups: metrics exporter hook (ENG-1310) to surface presubmit IO savings per team and add a presubmit-cache-priming job for long-lived branches.
Marco Liu: Nice approach — can we avoid adding another network hop in the critical path? Suggest the fetcher consult the runner-local cache first and only fall back to remote. (resolved in a1f4c2f)
Darren Kim: Request change: add an opt-out label for security releases so quiesce can't delay those merges. (addressed — added bypass flag and docs)
Priya Rao: Approve after CI green and a short observation window. Left a small nits about test durations.
Aisha Patel (author): replied — adjusted tests to use time.Sleep stubs and added a flag to shorten integration timeout in CI.
a1f4c2f: introduce lazy_fetcher implementation and unit tests
b3d6e10: wire presubmit workflow to fastpath and add conditional setup steps
c9e0b77: add quiesce adapter service + integration test harness
d2a8f55: docs, makefile targets and chart values for thresholds
Reduce presubmit network I/O by introducing a lazy fetch mechanism for large artifacts and add a conservative release quiesce adapter to reduce noisy automated rollouts. Default-off feature-flag; metrics and rollout docs included.
