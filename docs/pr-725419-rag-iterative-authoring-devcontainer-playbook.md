# RAG iterative authoring: Devcontainer playbook, assetized tool stubs, and local replay harness

Motivation: teams regularly iterate on retrieval-augmented-generation (RAG) flows and need a reproducible local authoring loop that mirrors Redwood runtimes (dev -> test -> smoke eval). This PR delivers a devcontainer playbook that wires a local Redwood sandbox with an incremental authoring loop, assetizes tool stubs (search, embeddings, extractor), and includes a small replay harness to capture and re-run user sessions for deterministic e2e checks.

What this PR contains (summary): - devcontainer/ with a prebuilt container image and task extensions that boot a lightweight Redis indexer, a local vectorstore shim, and a sandboxed Redwood client configured to point to localhost. - playbooks/iterative-authoring.md: step-by-step authoring guide with recommended prompts, metrics to watch (latency, token delta, retrieval hit-rate), and fallback strategies. - examples/tools/assetized/*: three tool stubs (search, embeddings, structured-extractor) wrapped as 'asset' bundles with manifest.json and small fixtures to exercise tool calling. - test-harness/replay: a deterministic session recorder and replayer (node script + fixtures) used in CI smoke to validate stable outputs for selected prompt sets. - telemetry/prometheus-rules and logging examples to surface drift/regression signals when iterating on retrieval or prompt templates.

Key motivations and design choices: - Local parity: the devcontainer mimics Redwood's routing and KV caching behavior (config knobs documented) so developers can observe batching + cache effects during iteration. - Assetized tools: packaging tool stubs as small asset bundles makes it straightforward to move from local prototyping to remote Hosted or Dedicated environments with the same manifest. - Deterministic replay: the harness records request/response pairs and seed values for RNG to enable stable smoke assertions in CI and local regression checks.

Testing performed: - Manual smoke runs in the devcontainer, exercising example authoring scenario and validating retrieval-to-generation handoff. - Added CI smoke pipeline (unit + replay) that runs on each PR to assert no regressions for the included prompt set. - Telemetry validated in a local Prometheus scrape and Grafana dashboard snapshot included in examples.

Checklist:
- [x] Devcontainer definition and tasks
- [x] Authoring playbook and guidance sections (prompts, metrics)
- [x] Assetized tool stubs with manifests and fixtures
- [x] Replay harness + example session fixtures
- [x] CI smoke pipeline integration
- [x] Docs cross-links to Redwood Console and Optimize guides

Risks and mitigation: - Environment drift: container image pinned to a small matrix of versions and a note added to playbook for upgrading. - Non-determinism in generation: harness records seeds and model version pins; when model variability is expected, the playbook documents tolerances and quality checks.

How to validate locally: follow playbooks/iterative-authoring.md; run ./devcontainer/tasks/start-local-sandbox.sh, then open examples/authoring/interactive.md; to run replay: node test-harness/replay/run.js --fixture fixtures/session-001.json.

Notes for reviewers: focus review on the playbook's operational guidance (search/caching knobs) and the asset manifest format—we intend this manifest to become the canonical minimal contract used by the Private installer and Dedicated asset import. CI pipeline included under .github/workflows/ci-rag-replay.yml and currently gated to the examples suite to keep runtime budgets low.
Samir Rao: Can we keep the replay fixtures small and representative?
Maya Thompson: Yes—two small sessions included; we intentionally kept payloads compact to keep CI cheap.
Elena Ivanova: Manifest schema needs 'runtime' and 'contract_version' fields.
Maya Thompson: Added contract_version and runtime entries to manifests and updated examples/README-authoring.md to document the fields.
Jonah Lee: Approved after CI green and small manifest tweaks.
Samir Rao: Left a minor nit about wording in the playbook; addressed with a follow-up commit.
