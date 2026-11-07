# hackathon: add internal LLM routing prototype

Motivation: for the internal hackathon we built a small routing microservice that demonstrates routing inference requests across two lightweight model variants (fast/cheap vs accurate/slow) based on simple latency and cost heuristics. This is a demo/prototype only and intended to seed a conversation about smarter routing policies that could later be integrated into the runtime.

What this PR does: adds a toy HTTP microservice (Python/Flask) that accepts Redwood-style request payloads, applies a configurable policy to choose a target model variant, forwards the request to a mock backend adapter, and returns annotated response metadata. Includes a README with run instructions, a small policy DSL (yaml) and unit tests for policy evaluation.

Key design notes: the service is intentionally minimal: policy evaluation is synchronous and in-process, there is no persistence, and backends are mocked by default. The routing policy supports rules based on max_latency_ms, cost_threshold, and explicit model_tags. This is NOT production-ready: no auth, no rate limiting, limited error handling.

How to run locally: python -m venv .venv && .venv/bin/pip install -r requirements.txt && FLASK_ENV=development FLASK_APP=app.py flask run --port 5003. See README for example curl payloads and expected annotated response.

Checklist: [x] prototype service and tests, [x] README and example payloads, [x] CI runs unit tests, [ ] integration w/real runtime (future).

Risks and follow-ups: this PR intentionally keeps dependencies tiny. If we iterate on this concept for product work we should extract policy evaluation into a library, add async evaluation to avoid blocking the request path, add metrics and tracing, and implement authentication/authorization and rate limiting.

Notes for reviewers: focus on clarity of policy DSL and test coverage rather than micro-optimizations. The implementation aims to be pedagogical for hackathon demos.
Diego Morales: Small nit on policy yaml parsing and suggest adding a test for missing keys.
Ava Chen: addressed and added test coverage for missing keys on 2024-11-14.
Priya Nandakumar: requested a short README section explaining why this is not production; added on 2024-11-15.
Demo prototype: internal hackathon artifact. Not intended for production use. Useful reference for routing policy ideas and early tests.
