# add-rag-tool-assertion-workshop-quickstart

Context and motivation: This quickstart provides a concise, opinionated workshop for embedding assertion-driven testing into RAG authoring workflows. Many teams struggle to verify tool-calling contracts and retrieval assumptions when iterating on prompt/tool designs. This PR adds a guided workshop with templates, runnable examples, and a CI smoke job so teams can validate contract correctness before shipping.

What this PR contains (single-pass summary): - New quickstart under quickstarts/rag-tool-assertion-workshop with step-by-step README, author checklist, and annotated examples. - Lightweight assertion schema (JSON) and a Python runner that loads assertions and replays a recorded scenario against local mock tool implementations. - Example tool contract templates for REST and function-tool bindings. - GitHub Actions workflow (ci/quickstarts-rag-tool-assertion.yml) to run smoke assertions on PRs. - Telemetry/telemetry-presets snippet to show recommended probe points for success/failure tagging.

Why this change: Improving authoring confidence is a high-leverage way to reduce regressions and production incidents when teams add new tools or refactor retrievers. The quickstart favors reproducibility (seeded examples and deterministic mock responses) and low setup friction (virtualenv + single command).

Checklist: - [x] Add README with learning goals and 30–45 minute workshop flow. - [x] Add assertion schema + examples. - [x] Add Python runner with CLI and fixtures. - [x] Add GitHub Actions smoke job. - [x] Add short docs site stub linking to quickstart (docs PR will follow). - [x] Tests: small unit tests and an end-to-end smoke run included in CI.

Commits (high-level): - 1a2b3c4: scaffold quickstart folder and README - 2d3e4f5: add assertion schema and templates - 3f4g5h6: implement python runner and fixtures - 4g5h6i7: add GH Actions smoke workflow and unit tests - 5h6i7j8: address review feedback (timeouts, better error messages)

Files changed (high level): - quickstarts/rag-tool-assertion-workshop/README.md (new) - quickstarts/rag-tool-assertion-workshop/assertions/schema.json (new) - quickstarts/rag-tool-assertion-workshop/examples/step_01_simple.py (new) - quickstarts/rag-tool-assertion-workshop/templates/tool_contract_rest.yaml (new) - ci/quickstarts-rag-tool-assertion.yml (new) - quickstarts/rag-tool-assertion-workshop/tests/test_runner.py (new)

Small illustrative snippet from the runner (for reviewers):

  # load assertion and replay (pseudo)
  assertion = load_json(path)
  response = mock_tool_call(assertion['input'])
  assert response['status'] == assertion['expected']['status']

Review conversation (abridged, chronological): - Liam Park (2026-02-12): "Nice workshop framing — can we make the schema explicit about which fields are required vs optional?" - Asha Gupta (2026-02-12): "Added schema comments and an examples folder to demonstrate optional fields." - Marisol Díaz (2026-02-13): "CI workflow looks good, but timeouts need tightening on the runner; I hit a flake locally." - Asha Gupta (2026-02-13): "Reduced default network timeout and added retry backoff. See commit 5h6i7j8." - Chen Wei (2026-02-13): "One nit: error messages from the runner should suggest a debugging command (e.g., --dump-on-failure)." - Asha Gupta (2026-02-13): "Added --dump-on-failure flag and updated README with troubleshooting section." - Marisol Díaz (2026-02-14): "Approved after the timeout and dump change; LGTM." - Liam Park (2026-02-14): "Approved; CI green on re-run"

CI summary: - Actions: build (pass), unit-tests (pass), quickstart-smoke (pass) — full workflow run on feature branch completed successfully after second push (fixed flakiness).

Merge outcome: Squash-merged into main on 2026-02-14 by Asha Gupta. Small follow-up TODOs captured as Linear tasks (ENG-4124 authoring-lint-rules; ENG-4125 add-docsite-integration).

How to try locally (README excerpt): - git clone && cd quickstarts/rag-tool-assertion-workshop - python -m venv .venv && source .venv/bin/activate - pip install -r requirements.txt - ./runner.py --scenario examples/simple --assertions assertions/simple.json

Notes for docs team: This quickstart is intentionally workshop-oriented and not a full replacement for the docs site example gallery; please link it from the RAG authoring page under "testing and assertions" and copy the minimal schema example into the docs short reference.
