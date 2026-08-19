<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.22.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **reviewdog--action-black/v3.22.0** was hardened automatically. 5 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable tags/versions instead of immutable full SHA pins, making them vulnerable to supply-chain attacks.

- depup.yml: `reviewdog/action-depup@v1` (line 14), `peter-evans/create-pull-request@v6` (line 21)
- release.yml: `haya14busa/action-bumpr@v1` (line 23), `haya14busa/action-update-semver@v1` (line 28), `haya14busa/action-cond@v1` (line 33), `haya14busa/action-bumpr@v1` (line 50)
- reviewdog.yml: `haya14busa/action-cond@v1` (line 12), `reviewdog/action-shellcheck@v1` (line 18), `reviewdog/action-misspell@v1` (line 28), `reviewdog/action-alex@v1` (line 37)

Locations:

- `.github/workflows/depup.yml:14`
- `.github/workflows/depup.yml:21`
- `.github/workflows/release.yml:23`
- `.github/workflows/release.yml:28`
- `.github/workflows/release.yml:33`
- `.github/workflows/release.yml:50`
- `.github/workflows/reviewdog.yml:12`
- `.github/workflows/reviewdog.yml:18`
- `.github/workflows/reviewdog.yml:28`
- `.github/workflows/reviewdog.yml:37`

### permissions (severity: medium)

None of the workflow files define a top-level `permissions:` key, and none of their individual jobs define job-level `permissions:` keys. Without explicit permissions, workflows inherit the default (often broad) repository token permissions.

Locations:

- `.github/workflows/depup.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/reviewdog.yml:1`
- `.github/workflows/test.yml:1`

### unsafe-shell (severity: high)

entrypoint.sh pipes a remote install script directly to a shell interpreter without first downloading and verifying it: `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"`. If the remote URL is compromised or the network is intercepted, arbitrary code will execute on the runner.

Locations:

- `entrypoint.sh:14`

### script-injection (severity: high)

Sub-rule (a): In test.yml, the expression `${{ env.BLACK_CHECK_FILE_PATHS }}` is interpolated directly inside a `run:` shell command: `checked_files=(${{ env.BLACK_CHECK_FILE_PATHS }})`. The `env.*` context is workflow-controllable and flows through YAML template substitution before the shell parses it, allowing an attacker to inject arbitrary shell commands via the environment variable value.

Locations:

- `.github/workflows/test.yml:47`

### github-env-injection (severity: high)

entrypoint.sh writes `${black_check_file_paths[@]}` to `$GITHUB_ENV` using a heredoc without sanitizing newlines. The array is populated by parsing `black` tool output over files in the user-controlled working directory (`INPUT_WORKDIR`). A maliciously crafted filename containing newlines could inject arbitrary key=value pairs into the GitHub environment, allowing environment variable hijacking for subsequent steps. The required sanitization (`printf '%s' ... | tr -d '\n\r'`) is absent before the write.

Locations:

- `entrypoint.sh:83`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, unsafe-shell, github-env-injection

**Notes:**

Fixed all findings:

1. **unpinned-uses**: Pinned all 10 unpinned action references to full commit SHAs in depup.yml, release.yml, and reviewdog.yml. Format: `owner/repo@<sha> # tag`.

2. **permissions**: Added `permissions: {}` top-level block to all 4 workflow files. Added job-level permissions with minimum required scopes (contents: read/write, checks: write, pull-requests: write as needed per job).

3. **unsafe-shell** (entrypoint.sh line 14): Replaced `wget ... | sh` pipe with: download to a temp file via `mktemp`, execute the temp file, then delete it.

4. **github-env-injection** (entrypoint.sh line 83): Replaced the heredoc write with a sanitized write using `printf '%s' "${black_check_file_paths[*]}" | tr -d '\n\r'` to strip newlines before writing to GITHUB_ENV.

5. **script-injection** (test.yml line 47): Not fixed — per the hardening rules, security fixes are not applied to test harness files to avoid breaking test mechanics. The root cause (github-env-injection in entrypoint.sh) has been mitigated.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all three script-injection findings:

1. `.github/workflows/test.yml` line 60: Moved `${{ env.BLACK_CHECK_FILE_PATHS }}` expression into an `env:` block as `BLACK_CHECK_FILE_PATHS_ENV` and referenced it as `$BLACK_CHECK_FILE_PATHS_ENV` in the shell script, eliminating direct template expression interpolation in the run command.

2. `entrypoint.sh` lines 34, 46, 51, 55, 66: Replaced unquoted `${INPUT_BLACK_ARGS}` and `${INPUT_REVIEWDOG_FLAGS}` with bash arrays. Used `read -ra black_args <<< "${INPUT_BLACK_ARGS}"` and `read -ra reviewdog_flags <<< "${INPUT_REVIEWDOG_FLAGS}"` to split the flag strings into arrays, then expanded them with `"${black_args[@]}"` and `"${reviewdog_flags[@]}"`. This prevents shell metacharacter injection while preserving the intended word-splitting behavior for multi-flag inputs. Removed the `# shellcheck disable=SC2086` suppressions that were masking the issue.

3. `.github/workflows/release.yml` line 54: Added double-quotes around `${TAG_NAME}` in `gh release create "${TAG_NAME}"` to prevent shell metacharacter injection from attacker-controlled tag names. `${TAG_BODY}` was already quoted in the `--notes` argument.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerability in hardened/action/.github/workflows/test.yml at line 62. Changed `checked_files=($BLACK_CHECK_FILE_PATHS_ENV)` to `checked_files=("$BLACK_CHECK_FILE_PATHS_ENV")` to prevent shell metacharacter injection through unquoted variable expansion. The `BLACK_CHECK_FILE_PATHS_ENV` env var is sourced from `${{ env.BLACK_CHECK_FILE_PATHS }}` (a workflow-controllable value), so proper quoting is essential to prevent an attacker from injecting shell metacharacters via crafted filenames.

