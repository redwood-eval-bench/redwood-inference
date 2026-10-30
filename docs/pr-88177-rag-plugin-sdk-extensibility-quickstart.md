# Introduce RAG plugin SDK + extensibility quickstart

Context: Customers often need to extend Redwood RAG flows with custom third-party adapters (custom vector connectors, proprietary tool wrappers, or edge-specialized preprocessors). The goal of this PR is to ship a minimal, well-documented RAG Plugin SDK and a hands-on quickstart that demonstrates how to author, test, and publish a local plugin that integrates with Redwood tool-calling and retrieval pipeline.

Motivation: Existing quickstarts show how to wire RAG flows end-to-end but don't provide a structured pattern for third-party plugin authors (type-safe lifecycle, capability discovery, and test harness). This makes it hard for partner teams and community contributors to build resilient adapters that behave correctly under streaming and fallback conditions.

What I changed: Added a new quickstart directory (examples/rag-plugin-sdk-quickstart) containing:
- A compact RAG Plugin SDK (TypeScript) that exposes a Plugin interface, capability descriptors, and a small adapter runtime for local testing. The SDK focuses on: typed request/response shapes, async streaming helper, and KV-prefetch hooks for retrieval optimization.
- Two example adapters: postgres-vector-adapter (shows paginated upserts and vector search shim) and pdf-tool-adapter (example tool that extracts, chunks, and annotates docs). Both include integration tests that run against a lightweight test double (sqlite in-memory for vectors, headless pdf fixture).
- Developer experience files: docs/quickstart.md (step-by-step), .devcontainer with docker-compose and a local redis/service mesh, a light GH Actions smoke-check workflow, and example npm scripts for build/test/watch.
- Docs site integration: added a short YAML entry (docs/site_nav.yaml) and the quickstart content (docs/rag-plugin-sdk.md) with runnable code blocks and an interactive sandbox link to the local devcontainer.

Checklist (author):
- [x] Add SDK and adapters
- [x] Add docs + site nav entry
- [x] Devcontainer and local test harness
- [x] CI smoke workflow
- [x] Integration tests for adapters

Commits (summary):
- feat: initial plugin-sdk and types + runtime helpers
- feat: postgres-vector-adapter with integration tests
- feat: pdf-tool-adapter and fixtures
- docs: quickstart guide and site nav entry
- ci: add workflow quickstart-smoke.yml
- chore: add devcontainer and docker-compose services
- fix: minor type errors discovered during review

Files changed (high-level):
- examples/rag-plugin-sdk-quickstart/README.md
- examples/rag-plugin-sdk-quickstart/sdk/index.ts
- examples/rag-plugin-sdk-quickstart/sdk/types.ts
- examples/rag-plugin-sdk-quickstart/adapters/postgres-vector-adapter/*
- examples/rag-plugin-sdk-quickstart/adapters/pdf-tool-adapter/*
- examples/rag-plugin-sdk-quickstart/tests/*
- docs/site_nav.yaml
- docs/rag-plugin-sdk.md
- .devcontainer/Dockerfile
- .devcontainer/docker-compose.yml
- .github/workflows/quickstart-smoke.yml

Review thread (highlights):
- Ben Walker: "Nice ergonomics. Can we make the capability descriptor immutable and include a version field for future compatibility?" — Aisha: "Agree, added descriptor.version and froze the object in sdk/index.ts — see last commit." 
- Priya Natarajan: "Integration tests look good, but the sqlite vector test doesn't exercise similarity scoring thresholds. Please add a negative-case test." — Aisha: "Added tests/vector_similarity_negative.test.ts and updated adapter to expose a confidence field in results." 
- Luca Moreno: "Docs look solid. Suggest a short troubleshooting section for common devcontainer issues and a note on credential handling for private DBs." — Aisha: "Added troubleshooting and a note about KMS-backed envvar injection for Private deployments."

CI summary: GitHub Actions matrix ran on node 18 and node 20; checks: build (pass), lint (pass with warnings fixed), unit (pass), integration (pass), docs-linkcheck (pass). One intermittent test flake in earlier run (sqlite lock) was addressed by serializing the integration tests in the harness.

Testing plan: Manual verification steps added to docs/quickstart.md (how to run devcontainer, run smoke workflow locally, run adapters against fixtures). Integration tests included in CI; maintainers can run npm run test:integration to reproduce.

Compatibility and risk: The SDK is purely in examples and has no runtime dependency on the Redwood serving runtime. It follows the same types used in the public SDKs (redwood-sdk-typescript) but is intentionally isolated so it can be copied by partners. No code in production packages was changed.

Follow-ups: (tracked in ENG-4389)
- Add a Java/Python reference implementation of the Plugin SDK
- Publish a partner template and npm package for the SDK
- Add e2e scenario that exercises plugin fallback when a tool times out

Merge outcome: Squash-merged to main after approvals from all reviewers. Back-ported the quickstart documentation link to the v2026.02 docs snapshot channel.
New quickstart: RAG Plugin SDK and example adapters (devcontainer, tests, docs). No breaking changes.
