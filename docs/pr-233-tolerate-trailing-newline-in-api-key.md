# Tolerate trailing newline in API key and improve auth error messaging

Context: SUP-4902. Customers reading API keys from mounted files often include a trailing newline. The Go SDK previously validated the API key string strictly and returned a generic 'invalid api key' error, causing confusion. Changes: if key ends with CR/LF, trim only those characters; if other whitespace exists, return a clearer error message pointing to the whitespace; add unit tests; update README guidance for Kubernetes secret mounts. Release: v0.6.2.
Hiro Tanaka: Any risk of accepting invalid keys?
Elliot Price: Only CR/LF at end; API still validates key server-side; this just prevents obvious footguns.
Jordan Blake: Please reference private deployment troubleshooting page.
Elliot Price: Added link in README.
