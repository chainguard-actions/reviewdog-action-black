<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.22.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **reviewdog--action-black/v3.22.2** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh downloads and pipes a remote install script directly to `sh` without first saving it to disk for inspection. The command `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"` fetches from a mutable URL (the `master` branch) and executes whatever content is returned, making this vulnerable to supply-chain attacks if the remote content is tampered with.

Locations:

- `entrypoint.sh:16`

### script-injection (severity: high)

Sub-rule (b): Multiple unquoted shell variable expansions of user-controlled inputs in entrypoint.sh. `${INPUT_BLACK_ARGS}` (from `inputs.black_args`) and `${INPUT_REVIEWDOG_FLAGS}` (from `inputs.reviewdog_flags`) are expanded without double-quotes in shell commands, allowing an attacker to inject shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) via those inputs. Affected lines include the `black` invocations and the `reviewdog` invocations in both the `github-pr-review` and `else` branches. The shellcheck `SC2086` suppression comments confirm the author was aware of the quoting issue but chose to suppress the warning rather than fix it.

Locations:

- `entrypoint.sh:31`
- `entrypoint.sh:43`
- `entrypoint.sh:47`
- `entrypoint.sh:50`
- `entrypoint.sh:57`

### github-env-injection (severity: high)

entrypoint.sh writes `${black_check_file_paths[@]}` to `$GITHUB_ENV` using a heredoc without any sanitization (`printf '%s' ... | tr -d '\n\r'`). The array is populated from `black_check_output`, which is derived from running `black` with the user-controlled `${INPUT_BLACK_ARGS}` argument. A crafted input could produce output containing newlines or the literal string `EOF` on its own line, breaking the heredoc delimiter and injecting arbitrary environment variables into subsequent steps.

Locations:

- `entrypoint.sh:75`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all three high-severity findings in entrypoint.sh:

1. unsafe-shell: Replaced `wget ... | sh` pipe with a two-step approach: download install script to a mktemp file, execute it, then remove it. This prevents supply-chain attacks from piping remote content directly to a shell.

2. script-injection: Parsed INPUT_BLACK_ARGS and INPUT_REVIEWDOG_FLAGS into bash arrays using `read -ra` and expanded them with `"${array[@]+"${array[@]}"}"` (handles empty arrays under set -u). Removed all SC2086 shellcheck suppression comments since quoting is now correct.

3. github-env-injection: Replaced the unsanitized heredoc write to $GITHUB_ENV with a sanitized loop using `printf '%s' "${_path}" | tr -d '\n\r'` for each path element, preventing newline injection that could break the heredoc delimiter or inject arbitrary environment variables.

