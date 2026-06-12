<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.22.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **reviewdog--action-black/v3.22.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh downloads and pipes a remote shell script directly to `sh` without first saving it to disk for inspection. The command `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"` fetches from a mutable URL (the `master` branch of an external repository) and executes it immediately, allowing a compromised or tampered remote script to run arbitrary code on the runner.

Locations:

- `entrypoint.sh:15`

### script-injection (severity: high)

Rule (b) violation: `${INPUT_BLACK_ARGS}` and `${INPUT_REVIEWDOG_FLAGS}` are expanded unquoted in shell commands. These variables are set from workflow-controllable inputs (`inputs.black_args` and `inputs.reviewdog_flags`) via the action.yml `env:` block. An attacker supplying values containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) can achieve command injection. The `# shellcheck disable=SC2086` comments on these lines confirm the unquoted expansions are intentional but they remain unsafe. Affected lines: `black --diff --quiet --check . ${INPUT_BLACK_ARGS}` (line ~27), `${INPUT_REVIEWDOG_FLAGS}` (line ~34), `black --check . ${INPUT_BLACK_ARGS}` (line ~43), `${INPUT_REVIEWDOG_FLAGS}` (line ~50), and `black --check . ${INPUT_BLACK_ARGS}` (line ~55).

Locations:

- `entrypoint.sh:27`
- `entrypoint.sh:34`
- `entrypoint.sh:43`
- `entrypoint.sh:50`
- `entrypoint.sh:55`

### github-env-injection (severity: high)

entrypoint.sh writes `${black_check_file_paths[@]}` to `$GITHUB_ENV` using a heredoc delimiter (`<<EOF`) without sanitizing the values. The array is populated from black formatter output (file paths in the checked repository). A repository file whose path contains a newline followed by `EOF` could terminate the heredoc prematurely and inject additional environment variable definitions into `$GITHUB_ENV`, potentially overwriting sensitive variables used by subsequent steps. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write.

Locations:

- `entrypoint.sh:76`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all three high-severity findings in entrypoint.sh:

1. unsafe-shell: Replaced `wget ... | sh` pipe with a two-step approach: download to a temp file via `mktemp`, execute separately, then delete the temp file.

2. script-injection: Split INPUT_BLACK_ARGS and INPUT_REVIEWDOG_FLAGS into bash arrays using `read -ra` and expanded them safely with `"${array[@]+"${array[@]}"}"` (handles empty arrays under `set -u` while preventing shell metacharacter injection).

3. github-env-injection: Replaced the raw heredoc write with a loop that sanitizes each file path using `printf '%s' "${_path}" | tr -d '\n\r'` before writing to $GITHUB_ENV, preventing heredoc delimiter injection via attacker-controlled file paths.

