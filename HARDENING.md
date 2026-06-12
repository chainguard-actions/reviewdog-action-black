<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.22.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **reviewdog--action-black/v3.22.4** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh downloads and pipes a remote install script directly to `sh` without first saving it to disk for inspection. The command `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/.../install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"` executes whatever content is served at that URL in a shell. Even though the URL includes a commit SHA in the path, the content is still executed without verification, and any MITM or CDN compromise would result in arbitrary code execution on the runner.

Locations:

- `entrypoint.sh:16`

### script-injection (severity: high)

Rule (b) violation: Multiple unquoted shell variable expansions of env vars that hold untrusted `inputs.*` values. In action.yml, `INPUT_BLACK_ARGS` is set from `${{ inputs.black_args }}` and `INPUT_REVIEWDOG_FLAGS` from `${{ inputs.reviewdog_flags }}`. In entrypoint.sh these are expanded unquoted — e.g. `black --diff --quiet --check . ${INPUT_BLACK_ARGS}` (line 30), `${INPUT_REVIEWDOG_FLAGS}` (line 37), `black --check . ${INPUT_BLACK_ARGS} 2>&1` (lines 42 and 46), and `${INPUT_REVIEWDOG_FLAGS}` again (line 53). An attacker-controlled input containing shell metacharacters (`;`, `|`, `$(...)`, etc.) can break out of the intended command and execute arbitrary shell commands.

Locations:

- `entrypoint.sh:30`
- `entrypoint.sh:37`
- `entrypoint.sh:42`
- `entrypoint.sh:46`
- `entrypoint.sh:53`

### github-env-injection (severity: high)

entrypoint.sh writes `${black_check_file_paths[@]}` to `$GITHUB_ENV` using a heredoc delimiter pattern (lines 78–80) without sanitizing newlines. The array is populated from black's stdout, which is influenced by the unsanitized `${INPUT_BLACK_ARGS}` (an attacker-controlled input). A crafted input could cause black to emit output containing newline characters that break out of the heredoc value and inject arbitrary environment variable assignments into `$GITHUB_ENV`, affecting subsequent steps in the calling workflow.

Locations:

- `entrypoint.sh:78`
- `entrypoint.sh:79`
- `entrypoint.sh:80`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all three high-severity findings in entrypoint.sh: (1) unsafe-shell: replaced `wget ... | sh` pipe with download-to-tempfile then execute pattern using mktemp; (2) script-injection: added double-quotes around all ${INPUT_BLACK_ARGS} and ${INPUT_REVIEWDOG_FLAGS} expansions (5 locations) and removed shellcheck disable comments that were suppressing the warnings; (3) github-env-injection: sanitized the black_check_file_paths array with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_ENV to prevent newline-based environment variable injection.

