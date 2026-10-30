# Implement KMS circuit breaker + fallback mode to prevent retry storms during KMS incidents

Add circuit breaker + fallback behavior around KMS calls to prevent cascading latency/5xx during KMS degradation. Operators will see explicit KMS_CIRCUIT_OPEN errors and new metrics for breaker state/transitions.
Context: During recent KMS throttling/timeouts (see SUP-19377), application-side retries amplified load and created a request storm against KMS, worsening tail latency and elevating 5xx. We’ve already standardized timeouts/retries in the KMS wrapper (PR #18421); this change adds circuit breaking + a documented degrade/fallback policy so we fail fast and shed load instead of stampeding KMS.

What this PR does:
- Adds a per-key (key ARN) circuit breaker in the shared KMS client wrapper with CLOSED/OPEN/HALF_OPEN states.
- Introduces a fallback mode policy for decrypt/envelope operations:
  - If circuit is OPEN: do not call KMS; attempt to serve from in-memory DEK cache if present (bounded TTL, non-persistent); otherwise fail fast with a mapped KMSUnavailable error (no retries).
  - Encrypt/new key generation paths: fail fast when OPEN (no cache fallback), returning a distinct error class to avoid queuing / retry loops.
- Caps concurrency to KMS with a small in-flight semaphore (prevents spikes during HALF_OPEN).
- Emits metrics + logs needed for on-call triage:
  - redwood.kms.circuit_breaker.state (gauge)
  - redwood.kms.circuit_breaker.open_total / half_open_total (counters)
  - redwood.kms.circuit_breaker.blocked_total (counter)
  - Structured log fields: kms.key_arn, kms.op, kms.error_category, kms.circuit_state, kms.circuit_open_reason
- Adds unit tests with fake clock to validate: state transitions, half-open probe behavior, and that OPEN short-circuits before any retry logic.
- Documents the operational behavior in-code (module docs + inline comment block) with explicit non-goals: we are not changing cryptographic primitives, not persisting plaintext key material, and not altering key rotation semantics.

SRE-requested changes included here:
- HALF_OPEN uses jittered probe scheduling to avoid synchronized thundering herd after an incident clears.
- OPEN duration is bounded; we re-check via controlled probes rather than staying OPEN indefinitely.
- Separate breaker keys for decrypt vs metadata calls (avoid a metadata blip taking down decrypt).
- Explicit error mapping for OPEN-short-circuit -> KMS_CIRCUIT_OPEN (distinct from timeout), and we do not apply caller retry policy to this class.

Config (defaults are safe for prod):
- kms.circuit_breaker.enabled=true
- kms.circuit_breaker.failure_threshold=8 (rolling window)
- kms.circuit_breaker.window=30s
- kms.circuit_breaker.open_cooldown=45s (+/- 20% jitter)
- kms.circuit_breaker.half_open_max_in_flight=2
- kms.circuit_breaker.cache_fallback_ttl=5m (decrypt only; bounded, memory-only)

Rollout notes:
- This is behind a feature flag kms.circuit_breaker.enabled (on by default in staging; gradual enable in prod per-region).
- Please coordinate with ENG-4828 (dashboard/alerts) before broad prod enablement.

Test plan:
- Unit tests for breaker transitions and short-circuit behavior.
- Staging: simulate KMS 429/timeout using the fault injection harness; verify: request rate to KMS drops sharply once OPEN, p95 latency stabilizes, and error responses are consistent and actionable.

Follow-ups (not in this PR):
- Update runbook playbook examples to include KMS_CIRCUIT_OPEN and recommended mitigations (ENG-4821 docs workstream).
- Extend integration tests to assert breaker behavior under sustained throttling (ENG-4825).

Screenshots / evidence:
- Added a short “expected graphs” note to help validate in staging: look for breaker_open_total increments and a drop in kms.request_total during simulated failures.
Sean Gallagher (SRE): The high-level approach looks right, but we need to guarantee that when the circuit is OPEN we do not accidentally enter the retry loop. Can you ensure OPEN short-circuits before retry middleware and emits a distinct error class so clients don’t retry?
Rafael Mendes (author): Yep. I moved the breaker check to the top of the wrapper call path (before retry/backoff). Added error KMS_CIRCUIT_OPEN and the retry classifier now treats it as non-retryable. Added tests to assert 0 KMS calls while OPEN.
Allison Grant: Metrics need to be consistent with our naming/tags. Please add key_arn as a tag but also cap cardinality. We’ve been burned by raw ARNs exploding series count.
Rafael Mendes (author): Good catch. I’m tagging by key_alias when present (normalized) and falling back to a hashed key_id label (first 12 chars) rather than full ARN. Full ARN is only in logs/traces.
Marcus Lin: Please confirm the cache fallback is memory-only and bounded TTL. We should not persist plaintext DEKs or extend lifetime beyond what’s already acceptable.
Rafael Mendes (author): Confirmed. This uses the existing in-memory DEK cache, TTL default 5m, and is only used for decrypt when circuit is OPEN. No persistent storage; no change to crypto primitives.
Ethan Park: For control-plane metadata calls (DescribeKey / GetKeyPolicy etc.), can we isolate the breaker so those failures don’t take down decrypt? We’ve seen IAM/permission issues on metadata while decrypt is fine.
Rafael Mendes (author): Implemented separate breaker keys per op class: decrypt/encrypt vs metadata. Also added separate metrics by op_class to help diagnose.
Nadia Rahman: CI integration-smoke failed once with flake in the fault injector. Please rerun and link the staging run output.
Rafael Mendes (author): Reran; green now. Linked the staging harness run in the check output; also reduced probe frequency in the test to avoid timing sensitivity.
Sean Gallagher (SRE): One more change request: half-open probes should be jittered; otherwise after a regional KMS blip every instance will probe at once when cooldown ends.
Rafael Mendes (author): Added +/-20% jitter to open_cooldown and randomized initial probe delay per process. Added a test that validates probe schedule uses jittered duration bounds.
Sean Gallagher (SRE): Approved with the jitter + non-retryable error mapping. This should materially reduce blast radius during throttling events.
Allison Grant: Approved after the tag cardinality fix.
