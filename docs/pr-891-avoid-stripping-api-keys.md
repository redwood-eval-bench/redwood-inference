# Avoid stripping API keys; only remove trailing CR/LF

Context: SUP-4821. Some customers store the Redwood API key in secrets managers or copy/paste flows that include a trailing newline. The Python SDK previously used .strip() on the key, which can remove leading/trailing whitespace unexpectedly and cause auth failures when keys are not exactly preserved. Changes: normalize by rstrip("\r\n") only; add a warning if whitespace is detected; add unit tests for newline and carriage-return cases; update docs string in README. Release: v0.9.4.
Connor OBrien: Does this break anyone who intentionally had spaces?
Ethan Park: We only remove trailing CR/LF now; if someone had leading spaces it would have failed anyway; we emit a warning when whitespace is present.
Aisha Rahman: +1, matches what we saw with Kiteframe. Please add note in release notes.
Ethan Park: Added.
