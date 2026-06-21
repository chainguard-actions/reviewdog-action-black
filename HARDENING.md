<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.23.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **reviewdog--action-black/v3.23.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh downloads and pipes a remote install script directly to `sh` without first saving it to disk for inspection. The pattern `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/.../install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"` executes whatever the remote server returns, making the action vulnerable to supply-chain attacks if the URL is compromised.

Locations:

- `entrypoint.sh:16`

### script-injection (severity: high)

Rule (b) violation: `${INPUT_BLACK_ARGS}` (sourced from `inputs.black_args`) and `${INPUT_REVIEWDOG_FLAGS}` (sourced from `inputs.reviewdog_flags`) are expanded **unquoted** in shell commands. Unquoted expansion allows word splitting and glob expansion of attacker-controlled values. Affected lines include:
- `black --diff --quiet --check . ${INPUT_BLACK_ARGS}` (unquoted INPUT_BLACK_ARGS)
- `black --check . ${INPUT_BLACK_ARGS} 2>&1` (unquoted INPUT_BLACK_ARGS, appears twice)
- `/tmp/reviewdog ... ${INPUT_REVIEWDOG_FLAGS}` (unquoted INPUT_REVIEWDOG_FLAGS, appears twice)
These should be double-quoted: `"${INPUT_BLACK_ARGS}"` and `"${INPUT_REVIEWDOG_FLAGS}"` (or use an array for multi-word flags).

Locations:

- `entrypoint.sh:28`
- `entrypoint.sh:40`
- `entrypoint.sh:44`
- `entrypoint.sh:50`
- `entrypoint.sh:59`

### github-env-injection (severity: high)

entrypoint.sh writes `${black_check_file_paths[@]}` to `$GITHUB_ENV` using a heredoc (`BLACK_CHECK_FILE_PATHS<<EOF`) without sanitizing newlines. The array is populated from `black`'s output, which is influenced by user-controlled `INPUT_BLACK_ARGS` (`inputs.black_args`) and by file paths in the repository. A crafted file path or black_args value containing a newline followed by `EOF` could break out of the heredoc delimiter and inject arbitrary environment variables into `$GITHUB_ENV`. The required sanitization (`printf '%s' ... | tr -d '\n\r'`) is absent.

Locations:

- `entrypoint.sh:76`
- `entrypoint.sh:77`
- `entrypoint.sh:78`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all three high-severity findings in entrypoint.sh:
1. unsafe-shell (line 16): Download reviewdog install script to /tmp/reviewdog_install.sh first, execute it, then remove it — no more pipe-from-internet-to-sh.
2. script-injection (lines 28, 40, 44, 50, 59): Added double quotes around all ${INPUT_BLACK_ARGS} and ${INPUT_REVIEWDOG_FLAGS} expansions; removed now-unnecessary shellcheck disable=SC2086 comments.
3. github-env-injection (lines 76-78): Sanitize the black_check_file_paths array with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_ENV; changed heredoc delimiter to BLACKEOF to reduce collision risk.

