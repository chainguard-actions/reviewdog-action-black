<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.25.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **reviewdog--action-black/v3.25.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh downloads a remote script and pipes it straight into a shell: `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/fd59714416d6d9a1c0692d872e38e7f8448df4fc/install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"`. The URL is pinned to a commit, but the content still goes to sh without being saved or checked first. Fix: download it to a file, verify its checksum, then run it as a separate step.

Locations:

- `entrypoint.sh:16`

### script-injection (severity: high)

Rule (b), unquoted expansion of untrusted data. The script expands variables taken from workflow inputs without double quotes: `${INPUT_BLACK_ARGS}` from inputs.black_args and `${INPUT_REVIEWDOG_FLAGS}` from inputs.reviewdog_flags. Offending lines include `black_check_output="$(black --diff --quiet --check . ${INPUT_BLACK_ARGS})"` and `${INPUT_REVIEWDOG_FLAGS} || reviewdog_exit_val="$?"`. Because the shell splits and glob-expands these values, a caller can inject extra arguments or file globs into black and reviewdog. Fix: read the flags into a bash array (e.g. `read -ra ARGS <<< "$INPUT_BLACK_ARGS"`) and expand it as "${ARGS[@]}".

Locations:

- `entrypoint.sh:31`
- `entrypoint.sh:44`
- `entrypoint.sh:49`
- `entrypoint.sh:53`
- `entrypoint.sh:64`

### github-env-injection (severity: high)

entrypoint.sh writes BLACK_CHECK_FILE_PATHS to $GITHUB_ENV using a fixed heredoc delimiter (`echo "BLACK_CHECK_FILE_PATHS<<EOF"`, then `echo "${black_check_file_paths[@]}" >> "$GITHUB_ENV"`, then `echo "EOF"`). The value is not sanitized first. It comes from black's output, which depends on file names in the repository (attacker-controlled in a PR) and on the workflow-controlled inputs workdir and black_args. Nothing like `printf '%s' ... | tr -d '\n\r'` is applied before the write, and the delimiter is a predictable `EOF`. A crafted file name could therefore end the heredoc early and add extra environment variables. Fix: sanitize the value and use a random delimiter.

Locations:

- `entrypoint.sh:89`
- `entrypoint.sh:90`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

All changes are in entrypoint.sh. Nothing has been run, so none of this is tested.

1. unsafe-shell: the reviewdog install script (still pinned to the same commit) is now downloaded to a temp file with mktemp. The script stops if that file is empty, runs it with `sh "$file" -b /tmp "$REVIEWDOG_VERSION"` (the old `-s --` is gone), then deletes it. It does not check a checksum, which the finding asked for: I had no reliable way to get the file's hash. The only protection is that the URL is pinned to a commit.

2. script-injection: INPUT_BLACK_ARGS and INPUT_REVIEWDOG_FLAGS are each split into a bash array with `read -r -d '' -a ... || true`, guarded by `[[ -n ]]`. This splits on whitespace (newlines included), like the old unquoted expansion did. The arrays are passed as ${arr[@]+"${arr[@]}"} so an empty array doesn't break under `set -u`. One small difference from before: the values are no longer glob-expanded. reviewdog still gets black's output on stdin from the echo pipe.

3. github-env-injection: BLACK_CHECK_FILE_PATHS is still one space-joined line, as before. Newlines and carriage returns are now stripped from it, and it is written to $GITHUB_ENV between random delimiters (16 bytes from /dev/urandom via od).

