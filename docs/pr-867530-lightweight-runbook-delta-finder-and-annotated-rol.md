# lightweight runbook delta finder and annotated rollback suggester

Context & motivation: On multiple recent incidents we spent too long reconciling runbook drift across services and rollouts. Small edits to runbooks (typos, step reordering) and correlated deploys are frequent causes of triage uncertainty. This PR introduces two lightweight helpers aimed at reducing time-to-diagnosis for on-call engineers: a runbook-delta finder (CLI + HTTP API) that computes compact, prioritized differences between runbook versions and surfaces the minimal actionable delta, and an annotated rollback suggester that correlates deltas with recent deploys and recommends candidate reverts with rationale and confidence heuristics. Also includes a 'dependency bisector' probe that runs quick side-effect-free checks to narrow down which deploy/manifest likely introduced behavior changes.

What changed (high level):
- New CLI: tools/rb-delta (Go) with flags to target services, timeframe, and scoring thresholds.
- HTTP handlers added to incident-bot API: /api/v1/runbook/diff and /api/v1/runbook/rollback-candidates.
- Small library: internal/runbook/diff (delta algorithm optimized for ordered step lists, uses a weighted LCS variant and semantic token mapping to compress no-op edits).
- Bisector: internal/bisect/quickcheck that executes idempotent smoke queries (via service probe registry) and suggests earliest failing deploy.
- Tests: unit + synthetic e2e, and an integration harness used in oncall-cases.
- Docs: docs/oncall/runbook-delta.md and runbook API doc examples.
- Telemetry: new metrics (rb_delta.duration_ms, rb_delta.candidate_count, bisect.steps) and traces wired to existing request span.

Commits (summary):
- 9a7b4c2 Add runbook diff library and basic comparator
- c1d2f3e Implement rb-delta CLI and test harness
- 33f0b4a Add API endpoints + request validation
- a4b9d6f Implement annotated rollback scoring and explanation payload
- bb2e11c Add quick bisector smoke probe + orchestration wrapper
- f0e1d22 Add docs, examples, and telemetry hooks
- ee7a0aa Fix race in cache eviction and add retries in bisector
- 0b19f17 Minor lint/test fixes, CI flake handling

Files changed (representative):
- tools/rb-delta/main.go (new)
- internal/runbook/diff.go (new)
- internal/runbook/score.go (new)
- http/handlers/runbook.go (modified/added)
- internal/bisect/quickcheck.go (new)
- telemetry/metrics.go (modified)
- docs/oncall/runbook-delta.md (new)
- tests/integration/rb_delta_e2e.test (new)

Design notes & heuristics:
- Delta algorithm favors minimal human-visible edits: step text similarity carries higher weight than reordering.
- Rollback candidates are assembled by correlating timestamped runbook commits, deploy manifests (via deploy-index), and bisector probe outcomes. Each candidate gets a "confidence" (0..1) and an explanation string indicating which signals matched (identical step removed, new env var, failed probe during bisect, etc.).
- The annotated rollback payload is intentionally conservative: suggestions are ranked but must be acknowledged by an oncall user before providing automated revert commands.
- Performance: the delta finder runs in <200ms for typical runbooks (<100 steps). Bisector default budget is 10 probes with 300ms timeout each; configurable per-call.

Backward compatibility & safety:
- API additions are additive; existing endpoints unchanged.
- No automated reverts are executed by default. The rollback suggester returns a command string that must be explicitly invoked by an authorized operator. RBAC checks added to new endpoints.
- Feature guarded behind 'runbook-delta' feature flag in config.

Testing & rollout plan:
- Unit tests cover diff and scoring heuristics with ~95% coverage for new packages.
- Integration e2e runs against a synthetic service image; results included in test artifacts.
- Canary rollout: enable feature flag for dev and staging first, then enable for a subset of production oncall teams, monitor metrics (time_to_first_hypothesis, rb_delta.candidate_accept_rate).

Review conversation (highlights):
- Maya Linden (reviewer): "Can we clarify the scoring explanation format? Also worried about exposing deploy IDs to low-privilege users." Author (Priya): "Added structured 'explanation' with redacted deploy IDs in low-priv roles; full ids require 'deploy-view' permission. Updated docs and unit tests to cover redaction."
- Ethan Cole: "Bisector runs occasionally caused load spikes in staging when probes are too chatty." Author: "Added probe budget, default timeouts, and probe registry whitelist. Bisector defaults to synthetic-safe probes only unless explicitly allowed."
- Noah Patel: "Consider emitting a compact trace span linking diff->bisect->candidate for easier RCA." Author: "Added span instrumentation and sample traces under telemetry rb_delta.trace_id."
- Maya: "Please add more e2e scenarios with reordering-heavy runbooks." Author: "Added two additional e2e scenarios covering step reordering and insertion; results in test artifacts." Reviewer approvals: Maya (approved), Ethan (approved after changes), Noah (requested minor doc tweaks -> addressed and approved).

CI & checks:
- go-test: passed (all new unit tests + existing ones)
- golangci-lint: passed after minor fixes
- integration/e2e (staging harness): initial run failed due to flaky probe in harness; rerun succeeded.
- security-scan: passed (no new secrets, allowed deps)
- deploy-preview: generated and reviewed. Final CI status: pass.

Manual QA notes (from SRE rotation):
- On staging we validated runbook diffs between v1.11 and v1.12 for 'payments' and confirmed the delta highlighted a removed retry step that correlated with increased 5xxs. Bisector suggested the config change in payments-service v1.12 as highest-confidence candidate.
- Time to first plausible hypothesis improved from ~14m (median) to ~4m in our small pilot.

Merge outcome: Squash merged to main on 2025-02-03 by Priya Desai. Changelog entry added. Feature flag created in infra config. Post-merge tasks: create onboarding note for oncall, schedule a short hands-on session during SRE office hours, monitor rb_delta.candidate_accept_rate, and add an automated smoke test to the canary pipeline.

Notes for follow-ups:
- Improve natural-language explanations for suggestions (ENG-842 backlog).
- Add optional auto-revert policy under enterprise admin control (guarded by strict RBAC and multi-sig).
- Consider integrating with alert enrichment to surface rollback candidates automatically in high-severity incidents.
Introduces `rb-delta` (CLI + API) which computes small, prioritized runbook diffs across service versions and suggests annotated rollback candidates with confidence scores. Adds a fast dependency bisector helper to narrow suspect deploys. No breaking changes.
Merged after approvals; feature gated. Monitor metrics and refine scoring based on pilot feedback. Oncall docs added under docs/oncall.
