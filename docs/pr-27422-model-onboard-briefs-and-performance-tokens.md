# Add model onboarding briefs and heuristic performance tokens

Motivation: New onboarding flows require concise, machine- and human-readable summaries that capture model intent, hard guarantees (tokenization, max length), and a lightweight set of measured performance heuristics. This PR introduces 'onboard briefs' — a compact canonical summary format stored alongside model profiles — and a small 'performance token' bucket that encodes measured latency/throughput/kv-cache characteristics used by the registry and release-notes automations. Goals: 1) Provide a neutral brief that SDKs and console can surface without parsing full profiles. 2) Expose stable, comparable performance tokens to drive release-note highlights and automated candidate ranking for rollouts. 3) Plug into the changelog templater so release notes better surface regressions or improvements in perf across variants. What I changed (high-level):
- Add briefs/ directory format and validation: profiles/briefs/{model}.brief.json
- Add measured performance token extraction as part of profile ingestion: scripts/ingest/perf_tokenize.py
- Wire the brief and tokens into the release-notes generator: tools/release_notes/templateer.py
- Add unit/integration tests and sample briefs for a collection of open models under samples/briefs/
- Documentation and developer guide in docs/onboard-briefs.md
Checklist:
- [x] Design doc linked in Linear ENG-512
- [x] Brief JSON schema and validator added
- [x] Ingested tokens persist to profile metadata and are included in compact export
- [x] Release-notes templater consumes tokens and generates perf blurb suggestions
- [x] Tests (unit + lightweight integration) added and passing
- [x] Manual smoke of SDK surfacing (TS + Python)
The brief JSON (example): {"id":"vicuna-13b-v1","summary":"Instruction-tuned conversational model tuned for chat assistants; best-effort contextualization up to 4096 tokens","dominant_use_cases":["chat","assistant","search-rewrite"],"limits":{"max_input_tokens":4096,"recommended_max_completion_tokens":512},"perf_tokens":{"p95_latency_ms":"t<=120","throughput_rps":"b>=50","kv_cache_efficiency":"high"}}
Key implementation notes:
- Performance tokens are heuristic strings (operator-based buckets) designed to be stable across runner/quantization variants; they are derived from the model's profile measurement vectors and mapped to a small taxonomy (t<=X, b>=Y, high/medium/low).
- The validator will reject briefs that omit required fields or include inconsistent limits compared to the canonical profile; this prevents brief/profile drift.
- release_notes/templateer.py adds a new section "Performance highlights" when tokens indicate a meaningful delta vs baseline, and emits rationalized bullet suggestions that engineers can accept/modify.
Backwards compatibility: briefs are optional; existing profiles still work. The tokens are additive metadata and do not change serving behavior.
Testing and CI:
- Unit tests for validator and tokenizer; integration test that runs sample ingest and ensures templateer emits a perf blurb for a synthetic delta.
- CI flakiness note: the integration involving measured latency uses a mocked runner in CI to keep runs stable.
Migration plan:
- New onboard CLI will write a brief alongside profile uploads; pipeline backfill can be run to seed briefs for top 200 models.
- Release-notes UI change is behind a feature flag.
Related work: ENG-401 (profile canonicalization), PRs in model-registry for changelog automation (see ENG-768 notes).
Risks and mitigations:
- Risk: tokenization thresholds tuned poorly for new hardware; Mitigation: thresholds are kept in config and can be recalibrated with a follow-up.
- Risk: brief drift when profiles update; Mitigation: validator rejects mismatches and we add a CI gate for profile-brief consistency.
Next steps: telemetry to measure adoption of briefs in console and SDKs, follow-up to canonicalize token taxonomy across dedicated/private deployments.
Diego Morales: Can we make the perf token taxonomy deterministic across CI and local runs? Suggest persisting the taxonomy version in each brief.
Author (Asha): Added taxonomy_version to schema and persisted it during ingest. See commit 7418f1f.
Lina Chen: templateer is great — small nit: prefer a shorter perf blurb when throughput and latency both change; condensed wording suggested.
Author (Asha): Adjusted templater wording and added unit to cap combined-blurb length.
Ravi Kapoor: validator rejects too aggressively for models with legacy limits; added a 'lenient-mode' flag to allow staged backfills.
Author (Asha): Implemented lenient-mode and flagged default to strict in CI.
Lina Chen: Approving — tests green locally and in CI. Squash merge ok after minor docs tweak.
