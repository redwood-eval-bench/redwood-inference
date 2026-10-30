# introduce-unified-rag-starterkit-multilang-bindings-and-docsite-snippets

Adds a compact, opinionated RAG starter kit with ready-to-run TypeScript, Python, and Go examples; includes telemetry adapter shims and docs snippets for quick authoring.
Motivation: Customers repeatedly asked for a compact, multi-language RAG starter that is small enough to fork as a template but comprehensive enough to cover model selection, embeddings, caching, and observability. This PR introduces a unified starter kit that: 1) provides TypeScript, Python, and Go reference runners; 2) includes small telemetry adapter shims that wire into Redwood Console probes; 3) adds docsite snippets and quickstart exercises so users can get from zero to a running RAG demo in under 10 minutes.

What changed (high level): - new starter directory with language-specific runners and minimal infra (docker-compose + local index) - telemetry adapters that emit the same semantic events we use in the Console (session_start, retrieval_hit, model_call, latency_bucket) - compact README and step-by-step Quickstart docs added to the docs snippets directory - small helper library for typed adapter bindings (TS/Python/Go) to demonstrate cross-language contract patterns - CI smoke tests and a local e2e script to exercise streaming and embeddings.

Checklist: - [x] Add TS/Python/Go runner templates - [x] Implement telemetry shim and examples - [x] Add docsite snippet pages and quickstart guide - [x] Add CI smoke scenarios that run locally in GitHub Actions - [x] Add CHANGELOG entry and release notes draft - [x] Validate minimal resource configuration for local dev (docker-compose)

Testing notes: The branch contains a lightweight GH Action that runs the smoke scenario (index build + single query per runner). Initially the Python runner failed intermittently due to a race in the local index seeder; added a short retry loop and re-ran CI. All checks are passing at merge. Manual verification steps are listed in the Quickstart README and were validated on macOS and Linux in a dev VM.

Backward compatibility: No API surface changes to Redwood platforms. Everything here is sample code and docs; no production runtime changes. The typed adapter helpers are opt-in and live in the examples namespace.

Related: ENG-2142 (starter kit initiative), PM-1098 (quickstart conversion for docs site), SUP-4021 (customer ask from onboarding).
Ethan Park: Nice compact layout. A few nit comments on naming and one suggestion to move the adapter event names to a shared constants file so examples don't drift.
Nora Kim: Thanks — extracted event names into telemetry/events.ts and updated bindings across languages.
Marisol Vega: Confirmed the quickstart steps work on my Linux laptop. Suggest adding a short troubleshooting section for common Docker networking issues.
Nora Kim: Added troubleshooting notes and an FAQ section to quickstart README.
Jonah Reed: CI smoke looks good. One test is timing-sensitive; suggest increasing timeout in the e2e script to reduce flakes.
Nora Kim: Increased timeout and added limited retry; reran CI — green.
