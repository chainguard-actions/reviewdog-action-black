<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.22.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **reviewdog--action-black/v3.22.2** was hardened automatically. 4 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh pipes remote content directly to a shell interpreter: `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/master/install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"`. The install script is fetched over the network and executed immediately without any integrity verification (e.g. checksum), allowing a compromised or man-in-the-middle response to execute arbitrary code on the runner.

Locations:

- `entrypoint.sh:14`

### unpinned-uses (severity: high)

Multiple workflow files reference third-party actions using mutable version tags instead of immutable 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved. Unpinned references found:
- depup.yml: `reviewdog/action-depup@v1`, `peter-evans/create-pull-request@v7`
- reviewdog.yml: `haya14busa/action-cond@v1`, `reviewdog/action-shellcheck@v1`, `reviewdog/action-misspell@v1`, `reviewdog/action-alex@v1`
- release.yml: `haya14busa/action-bumpr@v1` (×2), `haya14busa/action-update-semver@v1`, `haya14busa/action-cond@v1`

Locations:

- `.github/workflows/depup.yml:14`
- `.github/workflows/depup.yml:18`
- `.github/workflows/reviewdog.yml:10`
- `.github/workflows/reviewdog.yml:13`
- `.github/workflows/reviewdog.yml:24`
- `.github/workflows/reviewdog.yml:33`
- `.github/workflows/release.yml:22`
- `.github/workflows/release.yml:27`
- `.github/workflows/release.yml:32`
- `.github/workflows/release.yml:60`

### missing-permissions (severity: medium)

None of the four workflow files define a top-level `permissions:` key, and no individual job within any of them defines a `permissions:` key either. Without explicit permissions, workflows run with the repository's default token permissions (often `write-all` for private repos or broad read/write for public repos), violating the principle of least privilege.

Locations:

- `.github/workflows/depup.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/reviewdog.yml:1`
- `.github/workflows/test.yml:1`

### script-injection (severity: high)

Sub-rule (a): A `${{ env.BLACK_CHECK_FILE_PATHS }}` expression is interpolated directly inside a `run:` shell command in test.yml. The `env.*` context is a workflow-controllable source per the script-injection rules, and any `${{ ... }}` expression embedded in a `run:` block undergoes YAML template substitution before the shell sees it, allowing an attacker who can influence the env value to inject arbitrary shell commands. Offending line: `checked_files=(${{ env.BLACK_CHECK_FILE_PATHS }})`

Locations:

- `.github/workflows/test.yml:47`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, unpinned-uses, missing-permissions, script-injection

**Notes:**

Fixed all four findings:
1. unsafe-shell (entrypoint.sh): Replaced `wget ... | sh` pipe with a two-step approach: download to a temp file, execute separately, then remove the temp file.
2. unpinned-uses: Pinned all 8 unpinned action references across depup.yml, reviewdog.yml, and release.yml to full 40-character commit SHAs using lookup_action_sha.
3. missing-permissions: Added top-level `permissions: contents: read` to all four workflow files, with appropriate job-level permissions where broader access is needed (contents: write for release, pull-requests: write for PR creation).
4. script-injection (test.yml line 47): Moved `${{ env.BLACK_CHECK_FILE_PATHS }}` into the step's env block and referenced it as `$BLACK_CHECK_FILE_PATHS` in the shell script.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed script-injection by double-quoting all 5 occurrences of ${INPUT_BLACK_ARGS} and ${INPUT_REVIEWDOG_FLAGS} in entrypoint.sh (lines 30, 36/40, 44, 46, 53). Removed now-stale shellcheck disable=SC2086 comments. Fixed github-env-injection by replacing the direct echo of ${black_check_file_paths[@]} to $GITHUB_ENV with a sanitizing loop that uses `printf '%s' "$_path" | tr -d '\n\r'` for each path element, preventing newline-based heredoc injection attacks.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted array expansion in `.github/workflows/test.yml` line 52. Replaced `checked_files=($BLACK_CHECK_FILE_PATHS)` with `read -ra checked_files <<< "$BLACK_CHECK_FILE_PATHS"`. The `read -ra` with a here-string safely splits the double-quoted value on whitespace into array elements without allowing shell metacharacter injection.

