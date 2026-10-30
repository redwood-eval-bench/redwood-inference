# interactive-rag-playground-with-plugin-snapshots-quickstart

Motivation: Provide a minimal, local-first RAG quickstart that demonstrates interactive tool-calling + replayable plugin snapshots. Many existing quickstarts focus on batch examples or ephemeral tool stubs; this adds a small browser playground that (1) wires a lightweight RAG orchestrator to tool plugins, (2) records plugin responses as snapshots for deterministic dev/eval, and (3) includes a session export/import format and CLI for replay-based tests.

What this PR does: Adds a new quickstart under quickstarts/interactive-rag-playground that contains React playground UI, a Node dev server adapter that proxies Redwood API calls (dev mode shim), two example plugins (web-search and calendar) instrumented to emit snapshot records, a tiny CLI tool to export/import sessions, and docs/usage with a stepwise tutorial and assertions for structured-output linting. Also updates docs site sidebars and the quickstarts index.

Key design points: snapshots are JSONL with context+request+plugin-response+meta. The CLI supports --replay-mode to stub plugin outputs for local e2e. Playground includes a developer toggle to record live snapshots or run in replay mode. The tutorials explain how to use snapshots for deterministic evaluation and integration tests.

Checklist: 
- [x] playground UI + core RAG adapter
- [x] plugin examples with snapshot hooks
- [x] session export/import CLI + tests
- [x] quickstart README with step-by-step setup and common troubleshooting
- [x] docs site entry in quickstarts list and sidebar
- [x] lightweight e2e smoke tests that run in CI

Testing notes: Local dev tested on macOS and Linux. CI runs unit tests and CI-playground smoke which replays a recorded snapshot. During review an intermittent e2e failure was fixed by increasing the server-side request timeout and tightening a mock timer in the CLI tests.

Why unique: The snapshot-first developer workflow helps reproduce integration problems from real tool responses and speeds iteration when building tool-calling agents. It also acts as a simple regression harness for the RAG/structured-output pipeline without requiring heavy infra.
Liam Nguyen (request change): The snapshot format needs stable plugin identifiers and a version field so we can evolve the recorder without breaking older fixtures. Please add schema_version and plugin_id fields to each record.
Maya Chen (reply): Added schema_version and plugin_id to snapshot records and updated the CLI parser to validate backwards-compatible versions. Added unit tests for version mismatch behavior.
Soraya Patel (comment): Docs look great. Please add a short note about security considerations when recording real API responses (PII, tokens).
Maya Chen (reply): Added a Security Considerations section to the README and a --scrub flag in snapshot-cli to optionally strip candidate PII fields before writing fixtures.
Alex Johnson (approval): Verified replay smoke locally and on CI. The timeout bump for dev-server is reasonable for local debugging; approve to merge.
