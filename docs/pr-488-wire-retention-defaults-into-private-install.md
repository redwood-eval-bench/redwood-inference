# Wire retention defaults into Private install + document purge job backing services

Title: Wire retention defaults into Private install + document purge job backing services

Context
Private customers (regulated workloads) are asking two recurring questions during security reviews: (1) what are the default retention behaviors when they install Redwood Private, and (2) what infrastructure is required to reliably enforce purges (scheduled jobs, storage/indexing dependencies, and verification signals). This PR updates the private installer to ship private-friendly defaults for retention + audit logging and adds a dedicated doc section describing purge job prerequisites.

Scope
- Set secure retention defaults in the installer Helm values (aligned to retention controls v1): conservative request/response log retention, longer audit log retention, and explicit “disabled logging” behavior where applicable.
- Enable purge job runner + recommended throttles by default (idempotent, bounded deletes) with opt-out in values.
- Document required backing services for purge enforcement (what is mandatory vs recommended) and what to validate post-install (dashboards/metrics).
- Document upgrade notes (how to migrate from older installs that lack retention sections in values.yaml).

Non-goals
- This does not implement TTL enforcement itself (handled in redwood monorepo + telemetry stores).
- This does not add legal hold UI/flows (control plane feature).

Key decisions
- Defaults are “secure by default” for Private: request/response logs are short-lived unless explicitly extended; audit logs default to longer retention to satisfy auditability expectations.
- Purge jobs are enabled by default but throttled to avoid DB pressure. Operators can tune batch size / max runtime.
- Installer docs treat purge backing services as first-class dependencies: if you don’t deploy them, retention will degrade (policy is accepted but enforcement can lag/fail).

Defaults (Private)
- audit_logs: enabled, default TTL 365d (configurable)
- request_response_logs: enabled, default TTL 30d (configurable; can be set to 0 to disable storage)
- traces: default TTL 14d (if trace storage enabled)
- metrics: default TTL 30d (if long-term metrics storage enabled)
- artifacts (eval datasets/report packages): default TTL 90d when stored in object storage (if compliance pack enabled)

Implementation notes
- Adds a `retention:` section to the installer chart values with explicit per-class TTLs and a single place to switch purge jobs on/off.
- Adds `purgeJobs:` tuning (schedule, concurrency, maxRowsPerRun, maxRuntimeSeconds) with safe defaults.
- Adds docs: “Retention & Purge Enforcement Requirements” (DB/index/object storage, job runner, permissions, monitoring)

Validation
- Install on kind + minikube profiles: chart renders with and without optional stores.
- Values schema validation (helm lint + kubeconform) updated for new fields.
- Smoke test: retention config is visible in control plane and purge job CronJobs are created when enabled.

Rollout
- Recommended to land before enabling retention controls by default in Private distributions. Backward compatible (existing values files remain valid; new defaults apply when fields absent).
- Private installs now ship with secure retention defaults aligned to v1 retention controls tiers (audit logs + request logs TTLs, verification metrics enabled).
- Installer docs now explicitly list required backing services (DB/index/object storage/cron runner) for purge jobs and verification.
Priya Natarajan (review): Can you cite where the default TTL numbers are coming from (PRD/overview) and ensure they match what Security wants to message externally?

Vivek Kulkarni (author): Added a short “Source of truth” note to the doc linking to PM-842 and the retention controls overview; also aligned request_response_logs default to 30d (was 60d) to match the current Security tier defaults for Private.

Marcus Lin (review): Two concerns: (1) enabling purge jobs by default could surprise operators if backing services aren’t installed; (2) we need to be explicit that retention policy acceptance != enforcement if stores are missing.

Vivek Kulkarni (author): Updated docs with a callout: “Retention config is accepted, but enforcement depends on backing services.” Also made purge jobs default-enabled but added a startup log + Helm NOTES that warns when purgeJobs.enabled=true but optional stores are not configured.

Sean Gallagher (review): Please keep default concurrency low and provide a max runtime. We saw backlog after maintenance (similar to incidents thread). Also add guidance on what metrics to watch to confirm purge is healthy.

Vivek Kulkarni (author): Set default `purgeJobs.concurrency=1`, `maxRowsPerRun=50_000`, `maxRuntimeSeconds=900`. Added a verification section listing ttl_lag_seconds, purge_failures_total, purge_rows_deleted_total and recommended alert thresholds.

Allison Grant (review): Can we avoid introducing high-cardinality metrics labels from the installer side? Also ensure docs point to dashboards rather than telling operators to build their own.

Vivek Kulkarni (author): No metric label changes in this repo; only toggles. Docs now reference the standard “Retention Controls Verification” dashboards (observability pack) and how to import them.

Kevin Osei (review): For compliance pack customers, we should mention object storage requirement (bucket lifecycle vs application TTL) and that report packages may include retention config export.

Vivek Kulkarni (author): Added a subsection for “Compliance reporting pack artifacts” describing object storage requirement, recommended bucket lifecycle policies, and note that exports can be bundled with audit log exports.

CI note: kubeconform failed on first run due to missing schema update for new `retention.artifacts.ttlDays` field. Fixed in follow-up commit; reran workflow.

Priya Natarajan (approval): Looks good now. Please ensure we add this to the next private installer release notes.

Vivek Kulkarni (author): Added release_notes field + updated docs/operations/upgrade-notes.md.
