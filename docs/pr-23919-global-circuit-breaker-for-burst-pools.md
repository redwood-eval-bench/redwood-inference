# Global circuit breaker for burst pools (fleet safety cap)

- Adds a global (fleet-safety) circuit breaker for shared burst pools per (region, hardware_class) that automatically disables burst admission when safety caps are exceeded.
- Adds Prometheus metrics and alert annotations to explain when/why burst was disabled and when it recovered.
- Adds runbook links and structured reason codes surfaced to routing/support tooling.
Why: As we enable Dedicated burst in more regions/hardware classes, we need a hard backstop that prevents burst demand from consuming too much of the managed fleet and impacting non-burst tenants. We already have per-tenant caps and fairness, but we were missing a global safety limiter that can shut off burst admission quickly under fleet saturation or telemetry anomalies.

What: Introduces a global circuit breaker in the burst admission controller that opens when fleet-safety caps are exceeded for a given (region, hardware_class, pool_id). When open, burst requests are denied immediately with a deterministic denial reason and a suggested retry-after. The breaker automatically closes after a configurable cool-down + recovery threshold.

Behavior / semantics:
- Scope: breaker is evaluated per (region, hardware_class, pool_id). No cross-region behavior.
- Inputs: (a) fleet telemetry: burst_pool_allocated_gpus, burst_pool_target_gpus, burst_pool_saturation_ratio, and telemetry freshness; (b) config thresholds in policy engine.
- Open conditions (all must be true unless noted):
  - saturation_ratio >= burst.global_breaker.open_saturation_ratio (default 0.92)
  - allocated_gpus >= burst.global_breaker.open_allocated_gpus_floor (prevents flapping on tiny pools)
  - telemetry_age_seconds <= burst.global_breaker.max_telemetry_age_seconds (if telemetry is stale, we fail closed for burst: open breaker)
- Close conditions:
  - saturation_ratio <= burst.global_breaker.close_saturation_ratio (default 0.85)
  - breaker has been open for at least burst.global_breaker.min_open_seconds (default 30s) to avoid oscillation
- Denial semantics: breaker-open is an immediate denial (no queuing) and is evaluated before fairness token buckets.
- Determinism: given the same snapshot (telemetry + config), the decision is deterministic; we include the breaker state and evaluated thresholds in the internal debug payload (redacted externally).

API / surfaces:
- Admission response includes: deny_reason=GLOBAL_CIRCUIT_OPEN, retry_after_ms (bounded), and a short message suitable for logs.
- Adds internal debug fields (behind internal auth) to explain: breaker_state, open_reason, telemetry_age, saturation_ratio.

Observability:
- New metrics:
  - redwood_burst_global_breaker_open{region,hardware_class,pool_id} (gauge 0/1)
  - redwood_burst_global_breaker_open_total{region,hardware_class,pool_id,reason} (counter)
  - redwood_burst_admission_denied_total{region,hardware_class,pool_id,deny_reason} includes GLOBAL_CIRCUIT_OPEN
- Alert updates:
  - Adds annotations to BurstPoolSaturation and BurstAdmissionDenials alerts with the breaker status and thresholds (open/close ratios, telemetry age) so on-call can quickly tell if denials are expected.
  - Adds a new warning-level alert: BurstGlobalCircuitBreakerOpen (pages only if open > N minutes AND non-burst p95 latency is regressing, handled by SLO gate elsewhere).

Safety / rollout:
- Default config is disabled until explicitly enabled per region/hardware allowlist (ties into existing feature-flag and rollout gating).
- The breaker can be forced open via emergency config override (same pathway as other burst kill switches) but there is no new manual toggle endpoint in this PR.

Test plan:
- Unit tests: breaker open/close transitions, telemetry-stale behavior, hysteresis, and retry-after bounds.
- Integration tests: simulated saturation spikes; verifies burst denials flip to GLOBAL_CIRCUIT_OPEN while non-burst routing remains unchanged.
- Staging: replayed a recorded burst spike trace; confirmed breaker opens and alert annotations populate.

Notes / follow-ups:
- We should tune default thresholds per hardware class once we have more beta data (H100 vs L40S behave differently under partial saturation).
- The Grafana panel for breaker state is added in a separate observability-pack PR; this PR only emits metrics and annotations.

Refs:
- Linear: ENG-4902
- Related: pr-23822 (burst metrics baseline), pr-23837 (feature flags / kill switch), pr-23988 (SLO-based gating)
- Runbook: Burst pool overload mitigation (Confluence) to be updated with breaker-specific guidance after merge.
Keiko Yamamoto: Can we confirm we fail closed for burst when telemetry is stale? I'd rather deny burst than risk over-admitting based on an old low utilization snapshot.
Peter Holtz (author): Yep. In this PR, telemetry_age_seconds > max_telemetry_age_seconds forces breaker open with reason=TELEMETRY_STALE. Added a unit test and included telemetry age in the debug payload + alert annotations.
Vanessa Ortiz: Thresholds: 0.92 open / 0.85 close seem reasonable for H100 but L40S pools have been smaller. Do we have an allocated_gpus floor to avoid flapping on tiny pools?
Peter Holtz (author): Added open_allocated_gpus_floor (default 8) to prevent flapping on small pools. Configurable per pool via policy engine; defaults are conservative and disabled by default until allowlisted.
Jordan Lee: Determinism concern: saturation_ratio is coming from telemetry which can change between retries. That's fine, but the decision should be deterministic for a single evaluation. Are we snapshotting telemetry once per request?
Peter Holtz (author): Yes—telemetry snapshot is fetched once at request start and passed through the decision pipeline; we don't re-read mid-evaluation. Added a comment and made the snapshot struct immutable after creation.
Mina Haddad: How is retry-after computed? We should avoid synchronized retry storms when breaker opens.
Peter Holtz (author): retry_after_ms is randomized (jitter) within bounds based on min_open_seconds; also capped at 2s for now. We can revisit once we see client behavior. Included in response only for breaker-open and tenant-cap denials.
Sean Gallagher: Alerting: please add annotations that explain the evaluated thresholds and telemetry age so on-call can quickly assess. Also make sure this doesn't page by itself unless sustained + correlated with SLO regression.
Peter Holtz (author): Added annotations for open/close thresholds + current saturation + telemetry age. The new alert is warning-only and doesn't page; paging remains gated by SLO regression policies handled separately.
Keiko Yamamoto: Requesting changes: please add a small integration test that verifies GLOBAL_CIRCUIT_OPEN is returned ahead of fairness logic, otherwise we could burn tokens unnecessarily.
Peter Holtz (author): Done. Added integration scenario 'breaker_preempts_fairness' and adjusted ordering so breaker evaluation happens before token consumption. Pushed update.
Keiko Yamamoto: Approved.
Vanessa Ortiz: Approved (ship it; we'll tune thresholds during beta).
Sean Gallagher: Approved. Please link the runbook update PR once it's ready.
