# Add PLG Signal schema + event contract (v1) with validation + versioning

Summary: This PR introduces the canonical PLG-to-sales “Signal” schema (v1) and the upstream event contract used by the aggregation/scoring pipeline. It includes protobuf + JSON schema definitions, a small validation library, and versioning conventions so downstream systems (aggregator + HubSpot router) can evolve safely. This is the implementation referenced by PM-482 and ADR: usage/intent signals schema v1.

Context / motivation: We need a stable, privacy-aware contract for “sales-worthy” product-led signals (threshold + intent) so that: (1) telemetry/analytics can aggregate reliably across daily/weekly windows, (2) dedupe behavior is deterministic across backfills, and (3) routing systems can operate on a consistent set of fields/enums without embedding product-specific logic in every consumer.

What’s included: - New schema package: protos/plg/signal/v1/signal.proto defining Signal, SignalEvidence, SignalWindow, SignalType, SignalSeverity, and standard dimensions. - JSON schema mirror (for systems that can’t consume protobuf directly yet): schemas/plg/signal/v1/signal.schema.json. - Event contract for producers (telemetry + docs instrumentation): protos/plg/signal/v1/signal_event.proto and docs/telemetry/plg_signal_event_contract.md. - Validation utilities: src/plg_signals/validate.ts (and Go equivalent for router service) that enforce required fields, allowed enums, and canonical “dedupe_key” construction. - Versioning strategy: embedded schema_version (e.g., “plg_signal/v1”), plus additive-only rules for v1 and a path for v2 (new message + new version string).

Key schema details (v1): - Signal is an aggregated object (output of aggregator/scorer), not raw telemetry. It is safe to send to HubSpot after filtering + minimization. - Evidence captures why the signal fired (e.g., token volume threshold crossed, repeated 429s, dedicated docs views) without including raw prompts or end-user PII.

Illustrative snippet (proto): message PlgSignal { string schema_version = 1; string signal_id = 2; string org_id = 3; string workspace_id = 4; string account_domain = 5; SignalType signal_type = 6; SignalSeverity severity = 7; SignalWindow window = 8; string dedupe_key = 9; double score = 10; repeated SignalEvidence evidence = 11; google.protobuf.Timestamp first_seen_at = 12; google.protobuf.Timestamp last_seen_at = 13; map<string,string> dimensions = 14; }

Dedupe strategy (important): We standardize dedupe_key as a stable hash of: org_id + (optional workspace_id) + signal_type + window.kind + window.start_at (truncated to day) + primary dimension (when applicable, e.g., model_family). This ensures idempotency for daily aggregation jobs and prevents re-notifying the same account multiple times for the same window. See docs/telemetry/plg_signal_event_contract.md for the exact string format and hashing algorithm.

Privacy / minimization notes: - Explicitly excluded: prompt text, completion text, user emails, IP addresses, request IDs, and raw URL query params. - Allowed: org/workspace identifiers, coarse usage metrics (tokens, spend, error counts), docs page category (not full URL), and coarse “feature interest” enums. - Any free-form “notes” fields were intentionally omitted to keep HubSpot payloads predictable.

Changed areas / files (high level): - protos/plg/signal/v1/signal.proto (new) - protos/plg/signal/v1/signal_event.proto (new) - schemas/plg/signal/v1/signal.schema.json (new) - src/plg_signals/validate.ts (new) - src/plg_signals/dedupe.ts (new) - docs/telemetry/plg_signal_event_contract.md (new) - docs/adr/usage-intent-signals-schema-v1.md (link + references; no ADR content authored here)

Commit summary: 1) “schema: add plg signal proto v1 + enums” 2) “schema: add JSON schema mirror + examples” 3) “validation: add dedupe_key builder + required field checks” 4) “docs: event contract + versioning rules”

Review notes / discussion highlights: - Priya Natarajan: Requested renaming window_type -> window.kind and clarified that window.start_at must be truncated to UTC day for daily jobs. Implemented in commit 3. - Mei Lin: Flagged ambiguity between org_id and account_id; requested we standardize on org_id (internal) plus optional account_domain for CRM matching. Updated field names and docs accordingly. - Nadia Rahman: Asked to avoid “intent_score” in v1 to prevent conflating evidence with scoring; moved to a generic score at the signal level and kept evidence as structured facts. - David Kim: Requested a stable dedupe_key spec that aligns with HubSpot “idempotency” for timeline notes/tasks; added explicit string format + hashing algorithm section, and added a test vector in validate.ts.

Open questions (not blocking / future work): - Whether we need separate dedupe behavior for “evergreen intent” signals (e.g., repeated security doc views) vs strictly windowed usage signals. ADR suggests windowed for v1; revisit in v2 with suppression windows handled in scoring/router. - Mapping from docs instrumentation to “docs_category” enums: currently defined as SECURITY | PRICING | DEDICATED | PRIVATE | COMPLIANCE. Elliot Price to confirm final taxonomy.

Testing: - Unit tests for dedupe_key builder and validator (JS + Go) - Lint/proto compilation in CI - JSON schema validation via ajv against example payloads

CI: ✅ build (monorepo) ✅ lint ✅ unit ✅ proto-compile ✅ schema-validate

Rollout plan: This is a foundational contract; no runtime behavior changes in production until the aggregation job (ENG-7732) starts emitting PlgSignal rows using this schema. Consumers should treat v1 as additive-only.

Merge: Squashed and merged to main.
Internal: Adds canonical PLG Signal schema + event contract (v1) for upcoming aggregation + HubSpot routing pipeline (PM-482).
