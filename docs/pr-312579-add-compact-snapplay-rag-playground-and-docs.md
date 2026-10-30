# add-compact-snapplay-rag-playground-and-docsite-walkthrough

Motivation: Add a compact, easy-to-run RAG playground ("snapplay") that demonstrates typed tool-calling, a tiny plugin runtime shim, and a concise docsite walkthrough tailored for quick onboarding. This is intended for engineers who want a minimal local environment to prototype tool-calling agents and to exercise telemetry probes before integrating with larger examples.

What this PR adds (high level):
1) examples/snapplay/ts-runner - lightweight TypeScript playground server (express + websocket) that hosts an in-memory index and a mini plugin adapter exposing two sample tools (search and file-read stub).
2) examples/snapplay/py-adapter - a Python adapter example that demonstrates calling an external sync tool and returning typed JSON results.
3) docs/rag/playgrounds/snapplay.md - walkthrough with step-by-step setup, expected outputs, and troubleshooting tips.
4) .github/workflows/snapplay-smoke.yml - CI smoke job that runs the TS runner in CI, executes a scripted scenario, and validates telemetry events via a small assertion harness.
5) telemetry/telemetry-presets.json - a tiny preset that emits a compact set of debug probes (latency, token-count, tool-invocations) used by the playground.

Implementation notes:
- The TS runner uses the existing SDK client and a small mock index (SQLite in-memory) to keep the example self-contained.
- Tool contracts are defined with a TypeScript types file (tool-contracts.ts) so the example shows contract-first tool calling.
- The Python adapter demonstrates cross-language tool invocation by exposing a simple HTTP shim that the TS runner calls during the scenario.
- Docs include a short troubleshooting matrix (ports, common env var mistakes, and how to enable verbose telemetry).

Checklist:
- [x] Playground runtime scaffold (TS + minimal deps)
- [x] Python adapter and example tool
- [x] Docsite page with screenshots and exact commands
- [x] CI smoke workflow and lightweight assertions
- [x] Telemetry preset and example event assertions

Backwards compatibility: the playground is standalone and does not change runtime libraries or production configs. Related: ENG-2023 (authoring playgrounds), PM-114 (quickstart content brief), small follow-up planned to add a Rust/WASM shim if the sandbox proves useful.
Ethan Ross: Can we make the telemetry output deterministic for CI runs? Suggested adding a seeded clock to the TS runner — done in followup commit and noted in docs.
Sofia Nguyen: Docs look good; please add an explicit section about env vars that differ on Mac vs Linux — addressed with a troubleshooting table.
Carlos Rivera: Lint flagged a missing export in tool-contracts.ts and a flaky sleep in the smoke harness; author updated to remove the sleep and export the type properly.
Maya Patel (author): Responded to comments, added deterministic timestamping for CI tests, and shortened the CI job to 3m cap to avoid long runners.
Ethan Ross: Approved after CI rerun (verified telemetry assertions pass)
Sofia Nguyen: Approved
