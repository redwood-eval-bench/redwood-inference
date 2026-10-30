# Update Hosted API performance docs to link benchmark methodology + interactive report

Context: As part of the public benchmark report launch, we want developers landing in Hosted API docs (especially performance + latency/cost FAQs) to quickly find (a) the interactive benchmark page and (b) the detailed methodology. This PR adds cross-links in the relevant docs sections, plus a short, legally-reviewed disclaimer block so readers understand what the numbers represent and how to interpret variance.

What changed: 1) Added a new "Benchmarks" callout to Hosted API performance docs that links to the interactive table and the methodology page. 2) Updated FAQ entries for "How should I think about latency?", "Do you enable caching/batching in benchmarks?", and "Can I reproduce these results?" with links to methodology and a brief "what we measured" summary. 3) Added consistent link anchors so Marketing can deep-link from the landing page and the report PDF.

Notes: Copy is intentionally conservative on comparative language and points readers to the methodology for exact workload definitions. We avoid framing the results as guarantees, and we explicitly note that performance depends on request shape, region, and traffic conditions.

QA: Verified links resolve in local docs build, checked no broken anchors, and validated sidebar nav ordering unchanged.

Screenshots: N/A (content-only).
Renee Sullivan (review): The callout is good, but can we add one sentence about "latency/$" being a derived metric and where to find the definitions? (Avoid sending folks hunting.)
Wei Chen (author): Yep—added a second sentence: "Latency/$ is computed from p50/p95 latency and normalized cost; definitions + exact formula are in Methodology." Linked directly to the metric definitions anchor.
Soojin Lee (review): Please keep wording consistent with message house: prefer "latency per dollar (latency/$)" on first mention, then "latency/$". Also add links in the Hosted API FAQ where we already answer "Why is my p95 higher than p50?"
Wei Chen (author): Updated first-mention phrasing and added links in the p95 vs p50 FAQ entry. Also aligned the disclaimer block with the landing page copy (no "guarantees").
Henry Cho (review): Can we ensure we’re not implying these are "Redwood default settings" if someone has custom batching/caching? Suggest: "Benchmarks were run with the settings described in methodology" rather than "with Redwood optimizations enabled".
Wei Chen (author): Good catch—replaced "Redwood optimizations enabled" with "settings described in the published methodology" and removed the ambiguous phrasing.
Michael Grant (Legal review): Please add "not a promise of future performance" language in the disclaimer. Also avoid "best" / "fastest" adjectives on any doc page unless tied to a specific, substantiated claim with scope/timeframe.
Wei Chen (author): Added "not a promise of future performance" sentence to the disclaimer include. Searched the touched pages; removed "fastest" from one FAQ answer (now "low latency"), and left comparative statements to the report/landing page with the approved disclaimers.
Michael Grant (Legal review): Approved with the current disclaimer and no superlatives. धन्यवाद. (typo from my keyboard—please ignore).
Wei Chen (author): Thanks—merged the last wording tweak and ready for final approval.
Renee Sullivan (review): Looks good now. Approving.
Soojin Lee (review): Approved. Please ping me when merged so I can update internal enablement docs with the canonical links.
