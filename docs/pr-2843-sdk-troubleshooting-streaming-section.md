# SDK docs: add streaming troubleshooting (timeouts, reconnect, idempotency) + cross-links to known issues

## Summary
Adds a dedicated streaming troubleshooting section to the SDK docs and links it consistently to the new Known Issues hub articles. The goal is to reduce Support back-and-forth by giving customers copy/paste-ready guidance for the most common streaming failure modes (timeouts, reconnect, truncated streams) and by standardizing our language around safe retries and idempotency.

This PR is part of the Customer Success & Support escalation playbook refresh / known-issues sprint work. It is intentionally customer-safe (no account names, no internal dashboards).

## What’s included
- New docs page: “Troubleshooting streaming” with:
  - Common symptoms + likely causes (proxy idle timeouts, client read timeouts, network path resets)
  - Recommended client timeout defaults (connect vs read) and keepalive guidance
  - Reconnect flow for SSE/streaming responses (retry with jittered backoff)
  - Safe retry rules + idempotency guidance (what is safe to retry and what isn’t)
  - Clear “When to escalate to Support” criteria + required info (request IDs, region, model, SDK version)
- Cross-links to Known Issues:
  - Known issue: Streaming timeouts and reconnect (workarounds)
  - Known issues index landing page
- Minor edits to Python/TS/Go streaming guides to point to the troubleshooting page (instead of repeating partial guidance).

## Rationale
Support tickets SUP-1842 / SUP-1854 (and multiple repeats) consistently stall on:
- Unclear separation of customer network/proxy timeouts vs Redwood-side disconnects
- Inconsistent retry advice (some customers retry mid-stream unsafely)
- Missing “what to send Support” context (request IDs, regions, model/version, whether HTTP/2 is in play)

This doc consolidates the guidance and keeps the workaround details in the Known Issues pages so Support can update mitigations without chasing SDK guide copies.

## Not included / follow-ups
- No changes to server behavior; see ENG-4822 (improve streaming disconnect error details) for runtime/gateway improvements.
- No customer-specific mitigations beyond what’s already published as a Known Issue.

## Screenshots
- N/A (text-only docs).

## Implementation notes
- The troubleshooting content lives under SDK docs so it’s discoverable from language-specific guides.
- The “When to escalate” section aligns with the playbook language (request IDs, time window, error strings, impacted region/model).

### Example snippet (docs)
"If you see `stream reset` / truncated output mid-generation: (1) retry the request from the beginning (do not attempt to resume partial tokens), (2) use exponential backoff with jitter, and (3) include the `x-request-id` from the failed attempt when contacting Support. If failures persist for >10 minutes in a single region or affect multiple regions, escalate per the Known Issue guidance."

## Checklist
- [x] Links verified (no relative-link rot)
- [x] Terminology aligned: “stream reset / truncated stream / disconnect”
- [x] Support-safe (no internal tooling references)
- [x] Reviewed by Support/SRE for accuracy on retry + escalation criteria

Related:
- PR #2841 (Known issues hub + initial workaround articles)
- PR #2842 (Support docs nav updates + escalation/SLA links)


Elliot Price (Docs): Please add a short callout that links to the Known Issues index, not just the streaming article. Also, avoid stating exact internal SLO numbers here—keep it symptom/impact-based.

Wei Chen (Author): Updated to include both links in the header and removed the SLO mention. “When to escalate” now references symptoms/time window and required context.

Owen Phillips (Support): The retry guidance needs to be explicit that retries should restart the request (don’t attempt to resume a partial stream). Also add required fields: region + model + SDK version + request IDs.

Wei Chen (Author): Added a “Safe retries” subsection with “restart request only” language and a bullet list of required info for Support.

Rafael Mendes (SRE): Two tweaks: (1) prefer ‘jittered exponential backoff’ wording; (2) make it clear that disabling HTTP/2 is a temporary workaround and only recommended if it matches the Known Issue article.

Wei Chen (Author): Incorporated both. The HTTP/2 mention is now a link-out to the Known Issue page with a ‘temporary workaround’ label.

Kira Thompson (Writer): Can we standardize terms: “stream reset” and “truncated stream” appear—add them to a small glossary at the top.

Wei Chen (Author): Added a short “Symptoms glossary” block.

Elliot Price (Docs): Approved.

Owen Phillips (Support): Approved.

Rafael Mendes (SRE): Approved.
