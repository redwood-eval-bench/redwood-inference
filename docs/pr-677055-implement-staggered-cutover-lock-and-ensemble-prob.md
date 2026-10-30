# Implement staggered cutover lock and ensemble probe throttle for smoother rollouts

Motivation: We see frequent small regressions during progressive traffic shifts when probes (synthetic and live) are correlated and bursty. This PR introduces two coordinated mechanisms to reduce rollout flakiness: a staggered cutover lock that gates traffic shift steps until probe ensembles reach convergence, and an ensemble probe throttle that adaptively jitters and paces probes across cohorts to avoid correlated probe bursts.

What this implements: 1) rollout.cutover_lock: a stateful guard that holds progression when the probe ensemble's statistical convergence metric (ensemble_convergence_score) is below threshold. The lock is staggered by cohort so rollouts across multiple regions/replica pools progress independently rather than in lockstep. 2) probe.ensemble_throttle: a low-latency throttle controller that applies jittered pacing and priority smoothing for high-frequency probes (tail probes, KV-primed probes) to reduce probe-induced amplification of transient service pressure.

Behavioral guarantees and safety: - By default the cutover lock is opt-in for canaries; we wire it into the default canary policy with conservative defaults (min_convergence=0.85, min_samples=200). - The throttle operates in permissive mode for 72 hours post-merge in hosted test clusters to validate behavior before we flip enforcement for production rollouts. - We add a fallback policy: if the lock is held longer than max_hold_time (configurable), the orchestrator will continue with a limited-step slow-roll that caps traffic delta per minute to avoid indefinite stalls.

Testing and verification: - Unit tests for lock state transitions and throttle factor calculation. - Integration tests with the synthetic-canary harness that simulate correlated probe storms; added deterministic scenarios where throttle reduces probe amplification and cutover lock prevents 30%+ regression spikes during shift. - Canary experiments in 'staging-usa' for 48h: observed 2.7x reduction in rollout-related 95p latency spikes across three runs.

Upgrade and rollout plan: 1) Merge to main. 2) Deploy to staging clusters with enforcement disabled (telemetry only) for 72h. 3) Enable enforcement in a single region canary for production (traffic cap 5%) for 6 hours. 4) Gradual enable by region with monitoring playbook if ensemble_convergence_score trends downward.

Related work items and references: Linear: ENG-12841 (canary orchestration improvements), ENG-12902 (probe stabilization). Design doc: internal link https://linear.app/redwood/notes/cutover-lock-design.

Checklist: - [x] Implementation (code + tests) - [x] Unit/integration tests - [x] Canary verification in staging - [x] Observability (metrics, dashboards, alerts) - [x] Documentation in runbooks
Asha: Can you explain how the ensemble_convergence_score is computed and whether it's sensitive to sample skew?
Evelyn: The score is a weighted similarity metric combining rolling-window KS divergence across cohorts and a stabilization factor that downweights short-lived spikes; weights are configurable in rollout_policy.yaml. We added unit tests that simulate skewed probe distributions.
Miguel: We saw one flaky integration run where the throttle factor overshot during a burst. Is there a safety clamp?
Evelyn: Yes — throttle factor is clamped [0.1, 1.0] and we added a leaky-integrator decay to prevent overshoot. Added integration case that reproduced the previous overshoot and confirmed clamp behavior.
Samir: Approve after two nits on docs (minor wording) were addressed.
Asha: Approved. Miguel: Approved after re-run on staging passed. Merge-by-squash requested.
Introduce staggered cutover lock (opt-in) and ensemble probe throttle. Adds new metrics: ensemble_convergence_score, cutover_lock_state, probe_throttle_factor. Config knobs: rollout.cutover_lock.enabled, rollout.cutover_lock.min_convergence, rollout.cutover_lock.max_hold_time, probe.ensemble_throttle.enabled. Default is telemetry-only enforcement for 72h post-merge.
