<!-- markdownlint-disable -->

# Hardening Report: reviewdog--action-black/v3.26.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **reviewdog--action-black/v3.26.0** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

entrypoint.sh, the script the composite action's run step calls, downloads reviewdog's install script and pipes it straight into a shell: `wget -O - -q https://raw.githubusercontent.com/reviewdog/reviewdog/fd59714416d6d9a1c0692d872e38e7f8448df4fc/install.sh | sh -s -- -b /tmp "${REVIEWDOG_VERSION}"`. The URL is pinned to a commit, but the install script then fetches release binaries at run time, and the content is run without first being saved to a file and checked. The fix is to download to a file, verify its checksum, then run it separately.

Locations:

- `entrypoint.sh:16`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell

**Notes:**

entrypoint.sh no longer pipes reviewdog's install.sh into a shell. It now saves the script from the same pinned commit URL to a mktemp file and stops if the file is empty. Then it runs `sh "$file" -b /tmp "$REVIEWDOG_VERSION"`, with the pipe's `-s --` dropped so the script gets `-b` as its first argument, and deletes the file. The finding asked for a checksum check on install.sh itself. I didn't add one because I had no way to get the real hash, and making one up wasn't an option. The URL is pinned to a commit, and the install script is expected to check the release archive against the release's checksums.txt, but I didn't read install.sh to confirm that. If a later analysis pass still requires a hash for the script, this finding may come back. Separately, I changed the fixed 'EOF' heredoc delimiter used when writing BLACK_CHECK_FILE_PATHS to $GITHUB_ENV to a random one from /dev/urandom, because a file name containing a line 'EOF' could otherwise inject extra variables.

### Iteration 2

**Fixes applied:** unsafe-shell

**Notes:**

entrypoint.sh: the reviewdog install.sh (URL pinned to a commit) is now downloaded with wget to a mktemp file. The script checks that the file is not empty and starts with a shebang, then runs it as 'sh "$file" -b /tmp "$REVIEWDOG_VERSION"'. '-s --' was dropped so the installer still gets '-b' as $1. Afterwards the temp file is deleted. No checksum was added: no tool here could fetch the file to get its real hash, and I didn't want to make one up. Nothing was run or tested.

