<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.24.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **reviewdog--action-black/v3.24.0** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh pipes a remote script directly to a shell interpreter: `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/.../install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"`. Even though the URL is pinned to a specific commit SHA in the path, the content is still streamed directly to `sh` without being saved and verified first. This is the classic unsafe-shell pattern.

Locations:

- `entrypoint.sh:16`

### script-injection (severity: high)

Sub-rule (b): Multiple unquoted shell variable expansions of workflow-controllable inputs in entrypoint.sh. `${INPUT_BLACK_ARGS}` (sourced from `inputs.black_args`) and `${INPUT_REVIEWDOG_FLAGS}` (sourced from `inputs.reviewdog_flags`) are expanded without double-quotes in shell command contexts, allowing an attacker to inject shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.).

Offending lines:
- Line 29: `black --diff --quiet --check . ${INPUT_BLACK_ARGS}`
- Line 38: `${INPUT_REVIEWDOG_FLAGS}` (unquoted positional arg to reviewdog)
- Line 43: `black --check . ${INPUT_BLACK_ARGS} 2>&1`
- Line 48: `black --check . ${INPUT_BLACK_ARGS} 2>&1`
- Line 57: `${INPUT_REVIEWDOG_FLAGS}` (unquoted positional arg to reviewdog)

All five expansions must be double-quoted: `"${INPUT_BLACK_ARGS}"` and `"${INPUT_REVIEWDOG_FLAGS}"` (or use the guarded form `${VAR:+"$VAR"}` for optional inputs).

Locations:

- `entrypoint.sh:29`
- `entrypoint.sh:38`
- `entrypoint.sh:43`
- `entrypoint.sh:48`
- `entrypoint.sh:57`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed entrypoint.sh:
1. unsafe-shell (line 16): Replaced `wget ... | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"` with download-then-execute: download to a mktemp file, run `sh "${INSTALL_SCRIPT}" -b /tmp "${REVIEWDOG_VERSION}"` (dropped the `--` which was the shell's option terminator), then remove the temp file.
2. script-injection (lines 29, 38, 43, 48, 57): Tokenized list-style inputs `INPUT_BLACK_ARGS` and `INPUT_REVIEWDOG_FLAGS` into bash arrays using the xargs/NUL-delimiter pattern with `if [ -n "$VAR" ]` guards. Replaced all five unquoted expansions with proper array expansions `"${black_args[@]}"` and `"${reviewdog_flags[@]}"`.

