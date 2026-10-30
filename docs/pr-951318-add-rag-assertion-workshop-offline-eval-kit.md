# Add RAG assertion workshop and offline eval kit

Motivation: Engineers building retrieval-augmented agents need reproducible, offline workflows to author prompt/knowledge assertions and validate regressions without spinning up hosted infra. This quickstart provides a compact workshop that demonstrates assertion-driven evals, example agent tool-calling patterns, and telemetry probes that map errors to actionable traces.

What this PR adds: - a guided workshop README with hands-on tasks and expected outcomes - a tiny assertion DSL and runner (authorable JSON/YAML assertion files) to express functional and quality checks - an offline eval harness that spins a local retriever + lightweight model shim for fast iteration - telemetry probe presets (span names, tags, and sampling hints) and a small dashboard template - CI smoke job to run the assertion suite on PRs so regressions are caught early

Key usage notes: 1) Clone the quickstart and follow the README at quickstarts/rag-assertion-workshop/README.md for a 20-minute workshop. 2) To run locally: sh scripts/run-local-eval.sh --index ./data/small-corpus.json --assertions assertions/examples.yaml 3) To add assertions: author a new entry in assertions/ with description, input, expected retrieval top-k, and a fuzzy-match quality threshold. 4) Telemetry: add TELEMETRY_ENDPOINT to env and use the provided probes.yaml to map spans to workshop steps.

Checklist: - [x] Add README + step tasks - [x] Implement assertion runner and examples - [x] Add telemetry presets and dashboard skeleton - [x] Add CI smoke job and local run script - [x] Wire minimal example agent with tool-calling stub - [x] Tests: unit tests for assertion runner, end-to-end offline smoke run

Related: ENG-3921 (authoring-first evals), PM-684 (workshop materials), SUP-1205 (customer request for offline regression checks). Docs preview: https://docs.redwood.com/quickstarts/rag-assertion-workshop

Notes on scope: Designed to be small and local-first — it intentionally uses a model shim to keep costs low for teams onboarding. The toolkit is extensible and includes pointers for switching to a production-backed retriever or Redwood-hosted models.
Adds a compact, local-first RAG workshop and offline evaluation kit. Includes step-by-step authoring guide, assertion DSL, telemetry probe presets, and CI smoke job to validate regressions before release.
Liam O'Connor: Nice compact workshop — can we add a tiny section showing how to upgrade the small-corpus index to a cloud index? Recommend a short note in README.
Maya Chen: Added a migration note and a link to the dedicated index guide in README (line ~120). Also included a short code sample for swapping the retriever client.
Priya Natarajan: Telemetry looks good; one nit about span naming conventions (use 'rag.workshop.step' instead of 'workshop.step').
Maya Chen: Updated probes.yaml to use recommended span name and added comment explaining the mapping. Thanks!
Evan Lopez: CI job passed locally and on CI, approving — test coverage for runner is adequate.
Repo Bot: ci/smoke-rag-eval succeeded on run #4 (flaky network retry enabled).
