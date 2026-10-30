# Internal ops docs: Provision-to-ready readiness states, remediation, and dashboards/runbooks

Motivation: With the provision-to-ready automation rollout, we need a single, up-to-date internal operator guide that explains (a) what each readiness state means, (b) what to do when nodes are stuck or failing validations, and (c) where to look first (dashboards/logs/runbooks) without rediscovering tribal knowledge in Slack threads. This PR updates the internal docs to match ADR-017 readiness semantics and the current control-plane/Fleet Readiness Service behavior, and standardizes remediation guidance (drain/quarantine/reimage order) to reduce risk during on-call.

Summary of changes: Adds an operator-focused "Provision-to-ready operations" page that describes the canonical states (Allocated/Provisioning/Validating/Ready/Quarantined), the authoritative signals for transitions (agent heartbeat + validation artifacts + control-plane writes), and practical triage flows for common failures (kubelet join, driver parity, GPU health, NCCL smoke test, perf canary). Adds a "stuck in Validating" decision tree with time-based thresholds and clear ownership boundaries (Infra/Fleet vs Platform vs SRE). Links the docs to the current dashboards and runbooks, and includes a quickstart section for internal console usage (filters, drill-down, and where to find failure reasons/artifacts). Also documents the image channel rollout + rollback basics (candidate/stable), including the audit event trail for channel changes.

Notable clarifications: (1) Quarantine is a safety state used when failures repeat or indicate hardware/network fabric issues; it is not a generic catch-all for "slow provisioning". (2) Reimage is the preferred first-line remediation for image/driver drift failures; quarantine is preferred for GPU health or NCCL/fabric failures after retry policy is exhausted. (3) Do not manually set readiness state except via the approved operator workflow; state changes are audited.

Docs links added/updated in this PR: Provision-to-ready failures triage runbook; SLO/time-to-ready page; dashboard catalog; gameday plan; ADR-017 reference; Fleet Readiness Service API overview.

Rollout note: These docs assume the staged rollout feature flag is enabled only for us-east A100 pools; wording is careful to avoid implying global availability. References SUP-28433 for the "stuck in Validating" support pattern and INT-1042 for dashboard access/least-privilege.

Screenshots: Included updated internal console screenshots (redacted pool IDs) showing readiness state filters and failure reason drill-down.

Testing: Ran docs build locally and link checker; CI includes markdown lint + internal link validation.
Vanessa Ortiz (Infra/Fleet): "This is close. Two asks: (1) please make the remediation order explicit (drain -> quarantine -> reimage) and include the safe exceptions, and (2) the NCCL smoke test section should mention the known flake mode + when to re-run vs quarantine." Author (Talia Benson): "Updated remediation sections with a numbered flow, plus added a callout box for NCCL flake mitigation and the retry/backoff pointer. Also added the 'do not quarantine for slow provisioning' warning." Vanessa Ortiz: "Looks good now. +1."

Aisha Bello (SRE): "Can you add the actual SLI/SLO names we page on, and a link to the dashboard panel for P95 time-to-ready? Also, the 'page vs ticket' guidance should match our rollout tuning (ticket for elevated failure rate unless >X% for 15m)." Author: "Added the metric names used in alerts (time_to_ready_seconds p50/p95 + step_failure_rate) and linked to the dashboard section; updated guidance to match the initial tuning notes."

Ethan Park (Platform): "Minor: In the readiness semantics, please align wording with ADR-017: control plane stores current state; events/validation artifacts are attached but not event-sourced. Also clarify who can mutate state (service accounts vs operator overrides)." Author: "Aligned language, added an 'Authoritative state' paragraph, and added a table for read vs write permissions + audit expectations."

Connor O'Brien (Release Eng): "Please add a short section on rollout comms: where the go/no-go checklist lives and who signs off." Author: "Added a 'Staged rollout' section with checklist link and sign-offs; noted kill switch behavior." 

Fiona McCarthy (SecOps): "Thanks for calling out audit. Please add a line that image channel changes must be done via the approved workflow and are logged; no manual edits in Terraform state." Author: "Added explicit guidance and linked the audit events page."

Approvals: Vanessa Ortiz approved; Aisha Bello approved; Ethan Park approved; Connor O'Brien approved; Fiona McCarthy approved.
