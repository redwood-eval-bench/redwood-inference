# Fix header merge for streaming in Edge runtime (preserve Authorization)

Context: SUP-4884 (customer reports 403 when streaming from Next.js Edge). Root cause: in the streaming helper we constructed RequestInit twice and used object spread on headers; in Edge, Headers behaves differently and Authorization was dropped during the second merge. Changes: use a single canonical Headers instance; ensure Authorization is always set when apiKey is configured; add integration test using the edge fetch polyfill; update docs with edge runtime note. Release: v1.4.1.
Logan Wright: Can we reproduce without edge?
Dev Patel: Node undici path doesn't drop it; edge does due to Headers merge semantics.
Chloe Martin: Please keep workaround in ticket until release is cut.
Dev Patel: Release tag pushed.
