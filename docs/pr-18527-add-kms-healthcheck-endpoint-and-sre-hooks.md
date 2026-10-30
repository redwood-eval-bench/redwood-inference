# Add KMS dependency healthcheck endpoint + SRE hooks (predeploy gate)

Summary
- Adds an internal KMS dependency healthcheck endpoint for services that rely on envelope decryption.
- Exposes standardized SRE hooks so rollout tooling can gate deploys on KMS readiness and SRE can alert on sustained degradation.

Motivation / context
We’ve had a few cases (and a recent us-east timeout spike) where KMS latency/throttling started degrading request paths, but we didn’t have a single, low-noise signal that answers: “is KMS healthy enough for this service to safely take traffic right now?”

This PR implements an internal healthcheck that:
- does NOT call KMS on each request (avoids amplification / self-inflicted throttling)
- distinguishes transient vs sustained failures
- standardizes output so alerting + predeploy checks can consume it consistently

Design highlights
- Endpoint: GET /internal/health/kms
  - Returns 200 when KMS is healthy per local probe state
  - Returns 503 when probe has been failing beyond a configurable grace window
  - Includes machine-readable fields for automation (status, last_success_age_s, last_error_kind, last_error_message, probe_mode)
- Probe model: asynchronous background probe with caching
  - Runs every N seconds (default 30s)
  - Timeout per probe call (default 750ms) to avoid hanging the probe loop
  - Tracks last success timestamp + last failure reason (mapped via the canonical KMS error taxonomy)
  - Exposes a gauge for “kms_dependency_healthy” and counters for “kms_probe_errors_total{kind=...}”
- Probe modes
  1) decrypt_only (default for data-plane services)
     - Requires kms:Decrypt only
     - Decrypts a small, non-production probe ciphertext provided via config/secret (generated during provisioning)
     - Avoids requiring kms:Encrypt or DescribeKey permissions at runtime
  2) describe_key (opt-in)
     - Calls DescribeKey / GetKeyRotationStatus where permissions exist (control-plane / some private deployments)

What changed
- New internal route handler wired under existing internal mux: /internal/health/kms
- New KmsDependencyProber background task
- New config block:
  - kms.healthcheck.enabled (default true for control-plane; false for batch jobs)
  - kms.healthcheck.mode (decrypt_only|describe_key)
  - kms.healthcheck.interval (default 30s)
  - kms.healthcheck.timeout (default 750ms)
  - kms.healthcheck.fail_after (default 120s) (how long we tolerate sustained probe failures before reporting unhealthy)
  - kms.healthcheck.probe_ciphertext_ref (required for decrypt_only)
- SRE hooks:
  - Adds dependency status to the internal “/internal/sre/deps” payload (consumed by deploy tooling)
  - Adds a lightweight predeploy gate helper in the deploy-checks library that fails closed when KMS is unhealthy for > fail_after
- Observability:
  - Metrics:
    - redwood_kms_probe_latency_ms{mode,env,region}
    - redwood_kms_probe_errors_total{kind,mode,env,region}
    - redwood_kms_dependency_healthy{mode,env,region} (0/1)
  - Logs: structured log on state transition (healthy->unhealthy and unhealthy->healthy) to avoid log spam

Example response
200 OK
{"status":"ok","dependency":"kms","probe_mode":"decrypt_only","last_success_age_s":12,"last_error_kind":"","last_error_message":""}

503 Service Unavailable
{"status":"degraded","dependency":"kms","probe_mode":"decrypt_only","last_success_age_s":187,"last_error_kind":"Timeout","last_error_message":"context deadline exceeded"}

How to test
- Unit tests cover:
  - state transitions and fail_after grace behavior
  - error kind mapping -> endpoint output
  - probe loop jitter and timeout enforcement
- Manual:
  1) Start service with kms.healthcheck.enabled=true and a valid probe ciphertext ref
  2) Hit /internal/health/kms (should be 200 once the first probe succeeds)
  3) Break permissions / point to invalid key and confirm:
     - endpoint stays 200 for < fail_after
     - then flips to 503
     - logs one transition line

Rollout / safety
- Endpoint is internal-only (behind existing internal auth/mTLS) and not exposed publicly.
- Probe loop is rate-limited by interval; no per-request KMS call path.
- If probe is misconfigured (e.g., missing probe ciphertext), healthcheck reports “degraded” with a config error kind, but does not crash the service.

Follow-ups
- ENG-4828: wire these metrics into the KMS dashboards + paging thresholds (this PR provides the primitives).
- Private installer: provision/generate probe ciphertext artifacts automatically during install (tracked separately).
Sean Gallagher: Please make sure the endpoint doesn’t call KMS synchronously per request. We cannot afford to have alerting traffic amplify an incident.
Rafael Mendes (author): Confirmed — endpoint only reads cached probe state; probe loop runs on an interval with timeout + jitter. Added comment in handler to prevent future “quick fixes”.
Allison Grant: Can we add a gauge for healthy/unhealthy (0/1) in addition to counters? Dashboards are easier to build with a stable gauge.
Rafael Mendes (author): Added redwood_kms_dependency_healthy{mode,env,region} and documented semantics in code comment + README blurb.
Ethan Park: Decrypt-only probe requires a ciphertext. Where does it come from for Private? We should not require kms:Encrypt in runtime roles.
Rafael Mendes (author): Agreed. Default mode is decrypt_only and expects a probe ciphertext ref injected by provisioning. Added clear error when missing and linked follow-up for installer automation. No runtime kms:Encrypt required.
Kevin Osei: Please map the probe failure reasons to the canonical taxonomy (AccessDenied/NotFound/Throttling/Timeout/InvalidCiphertext) so on-call can filter consistently.
Rafael Mendes (author): Updated to use common KMS error_kind mapping and exposed last_error_kind in endpoint response. Added unit test for each mapping.
