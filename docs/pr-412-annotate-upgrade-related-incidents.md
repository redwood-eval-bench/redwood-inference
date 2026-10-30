# Annotate upgrade-window incidents + link to Private release artifacts

## Summary
When a Private upgrade is running (VPC or on-prem), we want incidents that fire in that window to be clearly labeled and automatically correlated to the exact release artifacts involved.

This PR updates incident-bot’s enrichment pipeline to:
- detect upgrade windows based on a canonical source (release train + installer upgrade state signals)
- tag incidents created during those windows as upgrade-related
- attach links to relevant release artifacts (release tag, build provenance, and installer plan bundle) so on-call can quickly determine whether a spike is likely upgrade-induced vs. product regression.

This is part of the "Private upgrade + rollback mechanism" workstream focused on auditability and reduced MTTR.

## Motivation
We’ve repeatedly had to manually correlate incidents during maintenance windows:
- Was there an upgrade running?
- Which Private version / chart bundle / installer build?
- Was the incident before/after DB migrations?

This PR makes that correlation automatic and consistent, and produces a trail that Support can paste into customer comms (esp. regulated customers).

## What changed
### 1) Upgrade window detection
We enrich incidents with `upgrade_window` metadata when *either* of the following are true:
- an upgrade window annotation exists in the release calendar (time-bounded)
- an installer upgrade-state heartbeat indicates an upgrade is in progress for the environment

Precedence:
1. installer upgrade-state heartbeat (most accurate, environment-specific)
2. release calendar window (fallback; useful when signals are delayed)

### 2) Artifact linking
If we can resolve a Private release version, we attach:
- `private_release_version` (e.g. `private-2025.05.1`)
- `release_artifact_url` (release page / provenance)
- `installer_bundle_url` (where the installer fetched the upgrade plan + checksums)

If we can resolve an upgrade plan, we attach:
- `upgrade_plan_id`
- `upgrade_plan_sha256`

### 3) Tag + annotation format (stable keys)
New normalized fields on the incident annotation:
- `upgrade_window:true|false`
- `upgrade_source:installer|calendar|unknown`
- `upgrade_phase:prechecks|backup|control_plane|data_plane|migrations|rollback|unknown` (best effort)

### 4) UI-friendly output
Incident-bot posts a short, skimmable block in the incident channel:

> Upgrade context: ✅ upgrade window detected (source: installer)
> Version: private-2025.05.1
> Plan: upgplan_01J9F3… (sha256: 7b9f…)
> Artifacts: release • bundle

## Implementation details
- Adds `UpgradeContextResolver` with two backends:
  - `InstallerStateBackend` (reads upgrade-state events emitted by the Private installer)
  - `ReleaseCalendarBackend` (reads the release train schedule exported to the annotations store)
- Adds `ArtifactLinker` to generate canonical URLs (release tag + bundle path) from version/build metadata.
- Adds a small cache (TTL 60s) to avoid hammering the annotations store during bursty incident creation.

## Changed areas
- `src/enrichers/upgrade_context.ts` (new)
- `src/enrichers/index.ts` (wire into enrichment pipeline)
- `src/backends/installer_state.ts` (new)
- `src/backends/release_calendar.ts` (new)
- `src/artifacts/linker.ts` (new)
- `src/renderers/slack_blocks.ts` (adds upgrade context section)
- `test/upgrade_context.spec.ts` (new)
- `test/fixtures/installer_state_events.json` (new)
- `docs/runbooks/incident-annotations.md` (updated)

## Example payload
Before:
- `tags: ["private", "k8s", "error-rate"]`

After:
- `tags: ["private", "k8s", "error-rate", "upgrade_window"]`
- `annotations.upgrade_window = "true"`
- `annotations.private_release_version = "private-2025.05.1"`
- `annotations.release_artifact_url = "https://github.com/redwoodinference/redwood-private/releases/tag/private-2025.05.1"`
- `annotations.upgrade_plan_id = "upgplan_01J9F3K8Q..."`

## Tests
- Unit tests for resolver precedence + partial data scenarios
- Fixture-based tests for mapping installer phases -> normalized `upgrade_phase`
- Integration smoke test in CI: creates a synthetic incident event, ensures Slack message includes upgrade section when signals are present

## Rollout / risk
- Safe by default: if upgrade context cannot be resolved, no new tag is applied and the renderer omits the upgrade block.
- No changes to incident creation; enrichment is additive.

## Follow-ups
- Add Grafana annotation emit for upgrade windows (so dashboards visually align with incident-bot tags) (tracked in ENG-4899)
- Consider adding a "maintenance window" tag for non-upgrade events (certificate rotations, infra migrations)

## Related
- ENG-4899 (auditability: upgrade artifacts and logs)
- SUP-18421 (upgrade-related incidents during migrations)
- Private upgrade/rollback project: change-management + runbooks

---

### Commits
- incident-bot: add upgrade context resolver + artifact linker
- incident-bot: render upgrade context in Slack + normalize tags
- tests: add fixtures and precedence cases
- docs: document new annotation keys

### CI
✅ lint
✅ unit
✅ integration-smoke

---

## Review notes
**Rafael Mendes:** Can we ensure we don’t mis-tag incidents when calendar window is broad (e.g., customer extends maintenance)?

**Allison Grant (author):** Added resolver precedence + tightened calendar matching to require environment id + explicit window. Also added `upgrade_source` so we can audit false positives.

**Connor O'Brien:** Please ensure artifact URLs are stable across mirror endpoints (on-prem).

**Allison Grant (author):** ArtifactLinker now prefers canonical release tag URLs; bundle link is only attached when we have a resolvable bundle origin. For on-prem mirrors, we attach `installer_bundle_ref` (opaque) rather than guessing a URL.

**Sean Gallagher:** Add rate-limit/caching; this will run during bursts.

**Allison Grant (author):** Added TTL cache (60s) keyed by env id + window start.

**Dev Patel:** Please add a short blurb to the Support playbook about what to paste into customer updates.

**Allison Grant (author):** Updated docs/runbooks/incident-annotations.md with a Support snippet + field definitions.

Approvals:
- Rafael Mendes ✅
- Connor O'Brien ✅
- Sean Gallagher ✅

Merged via squash.
- Incidents created during annotated upgrade windows now get an `upgrade_window:true` tag and (when available) `private_release_version`, `upgrade_plan_id`, and artifact links.
- Support/SRE can pivot from incident -> release artifact -> upgrade plan execution without manual correlation.
