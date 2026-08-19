<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.23.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **reviewdog--action-black/v3.23.0** was hardened automatically. 5 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh pipes a remote install script directly to `sh` without first downloading it to a file. The pattern `wget -O - -q https://raw.githubusercontent.com/.../install.sh | sh -s -- ...` executes whatever the remote server returns without any integrity check. If the remote URL is compromised or the response is tampered with in transit, arbitrary code runs on the runner.

Locations:

- `entrypoint.sh:16`

### script-injection (severity: high)

Sub-rule (a): The expression `${{ env.BLACK_CHECK_FILE_PATHS }}` is interpolated directly inside a `run:` shell command string. The value of `env.BLACK_CHECK_FILE_PATHS` is set by the action's entrypoint from file paths produced by `black`, which processes files in a caller-controlled working directory. Before the shell ever sees the script, GitHub Actions substitutes the raw expression value into the shell source, allowing shell metacharacters in the value to be interpreted. Offending line: `checked_files=(${{ env.BLACK_CHECK_FILE_PATHS }})`

Locations:

- `.github/workflows/test.yml:48`

### script-injection (severity: high)

Sub-rule (b): In the 'Release version' run step, the env vars `${TAG_NAME}` and `${TAG_BODY}` are expanded unquoted inside the shell command. `TAG_NAME` is sourced from `steps.tag.outputs.value` and `TAG_BODY` from `steps.bumpr.outputs.message` — both are workflow-controllable contexts. Unquoted expansion allows shell metacharacters (`;`, `|`, `&`, etc.) embedded in those values to be interpreted by the shell. Offending line: `gh release create ${TAG_NAME} -t "${TAG_NAME/refs\/tags\//}" --notes "${TAG_BODY}"`

Locations:

- `.github/workflows/release.yml:44`

### github-env-injection (severity: high)

entrypoint.sh writes `${black_check_file_paths[@]}` to `$GITHUB_ENV` using a heredoc without sanitizing newlines. The array is populated from `black`'s output, which processes files in the directory set by `INPUT_WORKDIR` (derived from `inputs.workdir` — a caller-controlled input). A malicious file path containing newline characters could inject additional environment variable assignments into `$GITHUB_ENV`. The required sanitization step (`printf '%s' ... | tr -d '\n\r'`) is absent before the write.

Locations:

- `entrypoint.sh:93`

### permissions (severity: medium)

None of the four workflow files define a `permissions:` key at the top level or at the job level. Without explicit permissions, workflows run with the repository's default token permissions, which may be overly broad (e.g., write access to contents, pull-requests, etc.). All four files are affected: depup.yml, release.yml, reviewdog.yml, and test.yml.

Locations:

- `.github/workflows/depup.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/reviewdog.yml:1`
- `.github/workflows/test.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection, permissions

**Notes:**

Fixed 5 findings across entrypoint.sh and 4 workflow files:

1. unsafe-shell (entrypoint.sh:16): Replaced `wget ... | sh` pipe with download-to-tempfile then execute pattern using mktemp.

2. script-injection (test.yml:48): Moved `${{ env.BLACK_CHECK_FILE_PATHS }}` into an `env:` block and referenced it as `${BLACK_CHECK_FILE_PATHS}` in the shell script.

3. script-injection (release.yml:44): Added double-quotes around `${TAG_NAME}` in the `gh release create` command to prevent unquoted shell expansion.

4. github-env-injection (entrypoint.sh:93): Replaced raw heredoc write with a sanitizing loop using `printf '%s' ... | tr -d '\n\r'` for each file path before writing to $GITHUB_ENV.

5. permissions (all 4 workflow files): Added minimal `permissions:` blocks — depup.yml and release.yml get contents:write + pull-requests:write; reviewdog.yml and test.yml get contents:read + checks:write + pull-requests:write.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerabilities in entrypoint.sh by converting unquoted variable expansions of INPUT_BLACK_ARGS and INPUT_REVIEWDOG_FLAGS into safe bash array expansions. Added `read -ra black_args <<< "${INPUT_BLACK_ARGS}"` and `read -ra reviewdog_flags <<< "${INPUT_REVIEWDOG_FLAGS}"` to parse the inputs into arrays, then replaced all unquoted `${INPUT_BLACK_ARGS}` and `${INPUT_REVIEWDOG_FLAGS}` expansions with `"${black_args[@]}"` and `"${reviewdog_flags[@]}"` respectively. This prevents shell metacharacter injection while preserving the intended behavior of passing multiple flags. All five affected locations (lines 31, 42, 47, 52, 61) have been fixed, and the `# shellcheck disable=SC2086` suppression comments have been removed.

