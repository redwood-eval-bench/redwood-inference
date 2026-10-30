# Improve offline bundle verification error messages + remediation hints

Installer preflight now emits actionable verification failures (error codes, likely cause, remediation, log paths). No change to verification semantics; still fails closed by default.
Context: With signed offline bundles, verification failures are expected to be one of the top support drivers during pilot. Today, failures often surface as generic crypto/library errors with no guidance on what to do next, and logs are not consistently referenced in the CLI output. This PR keeps fail-closed behavior unchanged, but makes the UX/support story significantly better.

What this changes: - Introduces a structured error taxonomy for bundle verification with stable error codes (RV-VERIFY-0xxx). - Maps common failure modes to a short 'what happened' + 'what to do next' remediation block, including where to find logs. - Adds a single-line summary suitable for copy/paste into tickets, plus a verbose output behind --verbose. - Normalizes log locations across Linux packages and containerized installer runs, and prints the resolved log path in the error footer. - Adds a 'transport integrity' remediation hint for hash mismatches seen in SUP-31892 (zip/unzip tooling, line-ending conversions, partial copies). - Adds time-skew specific messaging (not-yet-valid / expired) including suggested checks for air-gapped time sources.

Non-goals: - No change to verification policy (still blocks install on invalid/untrusted signatures). - No change to signature format / manifest schema (handled in other PRs). - No break-glass flag in this PR (tracked separately).

Testing: - Added unit tests for error mapping and formatting. - Updated integration tests to assert error codes + presence of remediation hints. - Manually validated output on Ubuntu 22.04 and RHEL 8 with tampered manifest + unknown key cases.

Docs: This PR updates the in-repo installer help text for the new error codes and log path footer. The full customer-facing docs update is tracked in ENG-4850.

Related PRs: - pr-24844-verify-bundle-signature-during-preflight - pr-24859-add-trust-store-and-key-rollover-support
Marcus Lin (review): The mapping from x509 errors to our codes looks good. Please make sure we never print raw certificate subjects/serials by default (avoid leaking customer environment details into screenshots/tickets). Also, suggest adding an explicit note that verification failures are non-retriable unless the bundle/trust store changes.

Ethan Park (author): Good call. I moved cert detail printing behind --verbose and redacted serial/subject in the default path. Also added a 'Retry only after…' line to the remediation templates (bundle re-copy, trust store update, clock fix).

Dev Patel (review): Love the copy/paste line. Can we standardize the last line so Support can grep it in support bundles? Something like 'redwood-private-installer: RV-VERIFY-0012'. Also include the log file path *and* the support bundle path if generated.

Ethan Park (author): Added a stable footer: 'redwood-private-installer: <CODE> (see <log path>)'. For support bundle path, we only have it after bundle generation; I added a hint: 'If you generated a support bundle, include support-bundle.tgz'. PR-25011 will wire verification status into the bundle automatically.

Owen Phillips (review): For hash mismatch remediation, please explicitly warn about 'text mode' FTP / CRLF conversion and using 'unzip -aa'. We saw exactly that in SUP-31892.

Ethan Park (author): Updated the RV-VERIFY-0021 remediation block with concrete do/don't examples and added a link target to docs/troubleshooting.md section.

Kira Thompson (review): Can we keep wording consistent with docs ("offline bundle", "manifest", "detached signature")? Also suggest changing 'signature invalid' to 'signature verification failed' (more precise).

Ethan Park (author): Updated terminology to match our docs draft and made the message text more precise.

Sahana Iyer (review): CI looks fine. One nit: error code ordering in errors.go is out of sequence and will be painful to maintain. Please sort by numeric code and add a brief comment on stability guarantees.

Ethan Park (author): Done. Sorted codes, added comment: codes are stable across minor versions and should not be reused.

Marcus Lin (final approval): Approved after redaction changes + code stability comment. Please ensure we track adding these codes to the external docs FAQ (ENG-4850).

Dev Patel (final approval): Approved. This will reduce back-and-forth on pilot tickets.
