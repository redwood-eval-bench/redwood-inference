# add contextual minidump collector and autoremediation hints for resolver workflows

Adds optional minidump collection for incident snapshots and a heuristic-based autoremediation hint generator for safe remediation suggestions. Backwards-compatible; opt-in via config.
Context and motivation: On recent incidents we found that responders spend significant time collecting and sharing local process dumps and contextual tracing snippets. This PR adds a lightweight, opt-in minidump collector that attaches sanitized per-request diagnostics to incident summaries and a compact autoremediation hint engine that surfaces safe, reversible suggestions (e.g. restart failing pod, increase connection pool, toggle feature flag) in the incident-bot UI and Slack alerts. The goal is not to automate fixes, but to reduce time-to-first-action for oncall by surfacing high-confidence, low-risk remediation hints and an attached diagnostic bundle.

Changes in this PR (high level):
- Add internal/collector/minidump.go: collects platform-appropriate minidumps for crashed or hung processes, applies a configurable sanitization filter, and packages into a compressed artifact.
- Add pkg/autoremediation/hints.go: lightweight rule engine that consumes incident signals (error fingerprints, recent restarts, pod resource metrics) and emits ranked hint suggestions with confidence and safe-revert commands.
- Integrate collector with resolver pipeline in cmd/incident-bot/resolver_handler.go: when an incident snapshot is created, if config.allowDiagnostics=true the bot will attempt a short-lived collector run and attach the artifact URL to the summary.
- Add UI metadata and Slack attachment formatting so hints and diagnostic links are shown in the alert thread without leaking secrets.
- Tests: unit tests for rule engine heuristics, e2e integration test that simulates a pod OOM and verifies a hint and diagnostic artifact are attached.

Checklist: [x] unit tests added [x] integration test added [x] feature gate and config docs [x] minimal telemetry added [ ] opt-in telemetry reviewed by privacy team (follow-up)

Security and privacy: collector runs under the incident-bot service account and only captures process-level stack summaries and symbolized frames; full memory contents are explicitly excluded. Artifacts are uploaded to the existing diagnostics bucket with ACL=private and pre-signed links valid for 24h by default. Config flags documented in docs/incident-bot/diagnostics.md.

Testing and validation: ran CI (see checks below). Local integration against a k8s kind cluster with an injected crash scenario verified artifact size stays under 2MB with default sanitization and hints matched expectations on 95% deterministic cases. Benchmark of collector latency shows median 320ms (cold), 50ms (warm) under typical workloads.

Rollout plan: feature gated behind incident_bot.collector.enabled=false by default. Gradual canary: enable for SRE team first, then 10% org, then full rollout. Backout: disable feature flag; artifacts remain in bucket but links are invalidated by changing presign TTL.

Related: ENG-1128 (diagnostics roadmap), ENG-1139 (oncall productivity OKR).
Marco Diaz: Can we avoid collecting raw addresses in the dump? Make sure sanitizer strips absolute paths and IPs. Also request: add a knob to limit artifact size.
Aisha Kapoor: Addressed in internal/collector/sanitizer_linux.go; added size cap and trimming behavior. See commit c4f2a10 and test coverage in rules_test.go.
Liang Chen: The rule engine looks sensible, but confidence scoring should prefer recent restarts over ephemeral spikes. Can we include a decay factor?
Aisha Kapoor: Added exponential decay parameter with default 3-hour half-life; exposed via config.autoremediation.decay_hours. Updated docs.
Sophie Martinez: Integration test flakes on CI occasionally due to network timeouts; please add retry and reduce external dependency surface (use local kind image).
Aisha Kapoor: Switched to kind harness and added a test helper to stub presign responses; fixed timeout and re-ran CI.
