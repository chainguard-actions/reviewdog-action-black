<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.22.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **reviewdog--action-black/v3.22.3** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh downloads a remote install script and pipes it directly to `sh` without first saving it to disk. This allows a compromised or man-in-the-middle'd remote server to execute arbitrary code on the runner. The offending line is: `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"`

Locations:

- `entrypoint.sh:16`

### script-injection (severity: high)

Rule (b): The env vars INPUT_BLACK_ARGS and INPUT_REVIEWDOG_FLAGS — set from `inputs.black_args` and `inputs.reviewdog_flags` respectively — are expanded unquoted inside shell command strings in entrypoint.sh. Unquoted expansion allows shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) embedded in those inputs to be interpreted by the shell, enabling command injection. The shellcheck SC2086 warnings are suppressed with disable comments rather than fixed. Offending lines include:
- `black --diff --quiet --check . ${INPUT_BLACK_ARGS}` (unquoted)
- `${INPUT_REVIEWDOG_FLAGS}` passed as unquoted trailing flags to reviewdog (two occurrences)
- `black --check . ${INPUT_BLACK_ARGS} 2>&1` (unquoted, two occurrences)

Locations:

- `entrypoint.sh:28`
- `entrypoint.sh:36`
- `entrypoint.sh:45`
- `entrypoint.sh:48`
- `entrypoint.sh:57`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed entrypoint.sh: (1) unsafe-shell: replaced `wget ... | sh` pipe with safe download-then-execute pattern using mktemp to save the install script to disk before running it; (2) script-injection: quoted all unquoted expansions of ${INPUT_BLACK_ARGS} (3 occurrences) and ${INPUT_REVIEWDOG_FLAGS} (2 occurrences) with double quotes, and removed the shellcheck SC2086 disable comments that were suppressing the warnings instead of fixing them.

