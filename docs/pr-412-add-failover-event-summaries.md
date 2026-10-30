# Add failover event summaries (trigger/rollback) with dashboard links

Context: As part of the Regional Failover Automation Upgrade, we now emit structured failover audit events from routing/control-plane (see redwood PRs: pr-19824-failover-event-audit-log.json, pr-19816-add-multi-region-slo-metrics.json). On-call feedback from rehearsals/game day planning is that the incident channel is missing a single, easy-to-skim summary of what the controller did and why, plus direct links to dashboards for verification (INT-1149: missing annotations/markers; SUP-25444: audit evidence requests). This PR enhances incident-bot to listen for failover automation events and post a concise message into the active incident channel (and #eng-sre for non-incident notifications), covering both trigger/execution and rollback outcomes.

What this PR does: - Adds a new event handler for failover automation audit events: FailoverRecommended, FailoverExecuted, FailoverRolledBack, FailoverSuppressed (dry-run / safety limits). - Formats a compact Slack summary with: region, route/model scope, action (flip/rollback), reason category, confidence + corroboration, and a short list of top evidence signals (reachability, 5xx burn, tail latency burn, capacity headroom). - Includes links: Region Health dashboard, Failover Signals dashboard, SLO burn panel, audit event viewer (internal), and the controller runbook entry point. - Adds message de-dupe + rate limiting (avoid spam during oscillation/noisy events); threads follow-ups under a single “Failover automation” root message per incident channel. - Adds optional “attention required” callouts when automation is blocked by approval hooks or per-tenant safety limits.

Notes on content/safety: - No tenant identifiers are included in Slack messages (only tier: hosted/dedicated and route group). - We intentionally cap evidence signals to top 3 items to keep the message readable; full details remain in audit logs.

Testing: - Unit tests for event parsing and Slack block rendering. - Integration test with a recorded event payload (from staging rehearsal) to ensure link construction works. - Manual verification in #eng-sre-sandbox with a synthetic event.

Rollout plan: - Land behind feature flag INCIDENT_BOT_FAILOVER_SUMMARIES (default off). - Enable in staging incident-bot first, then production before the next controlled failover game day.

Screenshots (Slack): Included in the PR conversation as pasted message examples.

Related: - ENG-2418 (metrics + burn alerts) - ENG-2416 (rollback guardrails) - Observability pack dashboards bundle (observability-pack PR-907) - Runbook entrypoint: routing-policy-failover-controller-runbook

Risk: Low. Changes are additive and gated; failure mode is missing a notification, not impacting routing.
Miles Thornton (review): Can we ensure we post both the initial recommendation AND the execution/rollback, but keep it threaded so we don’t blow up incident channels? Also, we should link to the Failover Signals dashboard by default, not the generic Region Health overview.

Devon Price (author): Yep. I added a root message keyed by (incident_channel_id, failover_event_group_id). Recommendation posts the root; execution/rollback replies in thread. Updated link priority order to: Failover Signals -> SLO burn -> Region Health.

Allison Grant (review): Please be careful about metric label explosion in the links; don’t include free-form reason strings as query params. Also confirm this doesn’t accidentally leak tenant IDs.

Devon Price (author): Good call. Removed raw reason text from URL params; we only include region + event_id (opaque). Tenant identifiers are not present in the incoming payload; we additionally assert/strip any tenant_id fields defensively.

Tomáš Novák (review): The rate limiting defaults look aggressive. During an incident we might miss key state changes. Can we rate limit only within the same state (e.g., repeated recommendations) but allow state transitions through immediately?

Devon Price (author): Updated: de-dupe window applies per (channel, route_group, from_state->to_state). State transitions (recommended->executed, executed->rollback) bypass limiter.

Rafael Mendes (review): Add a note to the on-call playbook that the bot message is informational and doesn’t imply the failover is complete; we still need validation steps. Also +1 to adding a runbook link.

Devon Price (author): Added a small disclaimer line in the Slack block and linked to the controller runbook section “Validate failover outcome”.

Approvals: Miles Thornton approved; Allison Grant approved after URL changes; Tomáš Novák approved after rate-limit tweak; Rafael Mendes approved.
Incident-bot now posts a concise Slack summary whenever regional failover automation triggers, executes, or rolls back, including primary signal evidence and links to relevant dashboards/audit logs.
Enabled flag in staging incident-bot. Production enablement scheduled after observability-pack dashboard bundle rollout completes (to avoid broken links).
