# runbook-replay-and-ephemeral-diagnostic-sandboxes

Motivation: On-call responders frequently need to re-run deterministic diagnostic steps against an incident's telemetry window. Today those steps are manual, error-prone, and often require produced artifacts that are hard to share with secondary responders. This PR introduces two coordinated features: (1) a /replay-runbook command that packages traces, metric snapshots, and log slices for a target incident window and exposes a canonical playback artifact, and (2) an ephemeral diagnostic sandbox that can execute small, hermetic diagnostic scripts against a replayed signal set so responders can iterate safely without touching production.

What this change does: - New bot command: /replay-runbook <incident-id|time-range> that collects a bounded window of signals (traces, metrics, logs) using existing telemetry exports and stores a playback bundle in the incidents bucket. The command returns a short-lived playback URL and an optional flag to boot an ephemeral sandbox. - Ephemeral sandbox: lightweight Kubernetes Job template (sandbox controller) that mounts the replay bundle and runs a configured diagnostics image (we provide builders for Node/Python/Go). The sandbox has strict network egress disabled by default and an RBAC gate so only authorized on-call roles can create them. - Playback API and adapter: a tiny adapter service mediates read-only access to the bundle and provides deterministic playback endpoints for the diagnostic scripts (replay HTTP endpoint, metrics server shim, trace collector shim). It also integrates a playback hook for the slo-toolkit exporter so SLI evaluations can run against replayed windows. - Observability & cleanup: the sandbox lifecycle is instrumented and emits breadcrumbs to the incident timeline. Teardown policy enforces TTL; orphaned sandboxes are garbage-collected by the controller.

Checklist: - [x] Basic collection and bundle serialization implemented - [x] Sandbox controller and Job template added - [x] RBAC checks (bot-level + k8s rolebindings) implemented - [x] Unit tests for adapter and RBAC gates - [x] E2E test that runs a simple diagnostic script against a replay bundle - [x] Security review and egress defaults applied

Why this is safe: we only allow read-only access to playback bundles, network egress is disabled, and sandbox images are pull-allowlisted via org policy. The playback adapter runs with limited privileges and only serves precomputed assets (no live ticketing or writeback). The feature is behind a feature flag and a per-org allowlist.

Related work: ENG-1573 (replay primitives), ENG-1621 (slo-toolkit playback hook). SUP-4421 contains a support case that motivated the RBAC gating and the TTL adjustments.

Notes for reviewers: the bulk of the diff is the new sandbox controller and the playback adapter; core bot command wiring is intentionally small and uses existing command parsing utilities. I split the e2e into a separate package so it can be exercised in CI independently.
Add /replay-runbook to capture and replay signal windows; ephemeral diag sandboxes that run diagnostic scripts against replayed traces; RBAC checks and cleanup policies. Backwards compatible.
Aisha Patel: Nice feature. Two questions: can we avoid storing PII in the bundle by default? Also request a small README for ops on how to manually garbage collect bundles.
Priya Raman: We redact PII by default (see serializer config) and added a doc fragment under docs/ops/replay-bundles.md. Marked in checklist.
Kenji Saito: sandbox teardown looked racy in local run; can you add a stronger leader election/owner reference?
Priya Raman: Added ownerRefs to the Job and tightened teardown ordering; pushed f8e9d0c to address the race.
Diego Morales: Security review notes: ensure the playback adapter does not allow query parameters that rehydrate live links. Also prefer a signed short-lived URL rather than direct object paths.
Priya Raman: Adapter now issues signed playback tokens (10m default). Updated tests to assert tokens expire.
Aisha Patel: Approve after CI green
Kenji Saito: Approve
Diego Morales: Approve
