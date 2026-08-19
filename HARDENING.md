<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.22.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **reviewdog--action-black/v3.22.3** was hardened automatically. 4 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh downloads and pipes a remote install script directly to `sh` without first saving it to disk for inspection. The command `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"` fetches content from a mutable URL (the `master` branch) and executes it immediately, allowing a compromised upstream to run arbitrary code on the runner.

Locations:

- `entrypoint.sh:16`

### script-injection (severity: high)

Sub-rule (a): A GitHub Actions expression `${{ env.BLACK_CHECK_FILE_PATHS }}` is interpolated directly inside a `run:` shell command string: `checked_files=(${{ env.BLACK_CHECK_FILE_PATHS }})`. The `env.BLACK_CHECK_FILE_PATHS` value is set by the action from black's output (which is influenced by user-controlled inputs such as `INPUT_BLACK_ARGS` and `INPUT_WORKDIR`). Injecting this expression directly into the shell command allows an attacker to embed shell metacharacters that are parsed before the shell ever sees the value.

Locations:

- `.github/workflows/test.yml:47`

### github-env-injection (severity: high)

entrypoint.sh writes `${black_check_file_paths[@]}` to `$GITHUB_ENV` using a heredoc pattern without sanitizing newlines first. The array is populated from black's stdout, which is influenced by user-controlled inputs (`INPUT_BLACK_ARGS`, `INPUT_WORKDIR`). An attacker who can influence black's output (e.g. via crafted file paths containing newlines) could inject arbitrary key=value pairs into the GitHub environment, potentially overwriting sensitive environment variables for subsequent steps. The required sanitization (`printf '%s' ... | tr -d '\n\r'`) is absent.

Locations:

- `entrypoint.sh:79`

### missing-permissions (severity: medium)

None of the four workflow files define a top-level `permissions:` block, and no individual job within any of these files defines a `permissions:` block either. Without explicit permissions, workflows run with the default token permissions (which may be `write-all` depending on repository settings), granting broader access than necessary. Affected files: depup.yml, release.yml, reviewdog.yml, test.yml.

Locations:

- `.github/workflows/depup.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/reviewdog.yml:1`
- `.github/workflows/test.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, github-env-injection, script-injection, missing-permissions

**Notes:**

Fixed all 4 findings: (1) unsafe-shell in entrypoint.sh: replaced wget|sh pipe with download-to-tempfile then execute pattern; (2) github-env-injection in entrypoint.sh: sanitized newlines using printf+tr before writing to GITHUB_ENV; (3) script-injection in test.yml: moved ${{ env.BLACK_CHECK_FILE_PATHS }} expression to step env: block and referenced as plain $BLACK_CHECK_FILE_PATHS variable; (4) missing-permissions: added minimal top-level permissions blocks to all 4 workflow files (depup.yml, release.yml, reviewdog.yml, test.yml).

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:

1. entrypoint.sh (lines 34, 39, 44, 47, 54): Replaced unquoted ${INPUT_BLACK_ARGS} and ${INPUT_REVIEWDOG_FLAGS} expansions with safe bash array splitting. Used `IFS=' ' read -r -a black_args <<< "${INPUT_BLACK_ARGS}"` and `IFS=' ' read -r -a reviewdog_flags <<< "${INPUT_REVIEWDOG_FLAGS}"` to split the inputs into arrays, then expanded them with `"${arr[@]+"${arr[@]}"}"` — a pattern that safely handles empty arrays while keeping each element properly quoted when non-empty. Removed the now-unnecessary `# shellcheck disable=SC2086` comments.

2. .github/workflows/release.yml (line 44): Added double quotes around `${TAG_NAME}` in the `gh release create` command, changing `gh release create ${TAG_NAME}` to `gh release create "${TAG_NAME}"`. TAG_NAME and TAG_BODY were already set via the env: block; only the shell expansion quoting was missing.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in .github/workflows/test.yml at line 54. Replaced the unquoted array expansion `checked_files=($BLACK_CHECK_FILE_PATHS)` (which allowed glob expansion and shell metacharacter injection) with `IFS=' ' read -ra checked_files <<< "$BLACK_CHECK_FILE_PATHS"`. This safely splits the space-separated file paths into an array while keeping the variable properly quoted. Also removed the `# shellcheck disable=SC2206` comment that was suppressing the shellcheck warning about the unsafe expansion.

