<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v2.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v2.0.0** was hardened automatically. 7 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct expression interpolation in run: blocks. Multiple steps in action.yml interpolate ${{ }} expressions directly inside shell commands:
1. 'Install Rust Problem Matcher' step: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — github.action_path is injected directly.
2. 'flags' step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — inputs.toolchain and inputs.components are interpolated directly.
3. 'rustup toolchain install' step: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — steps.flags.outputs.* are interpolated directly into a shell command.
4. 'Downgrade registry access protocol' step: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ...` — steps.versions.outputs.rustc-version is interpolated directly.
All of these allow an attacker-controlled or workflow-controlled value to be injected into the shell before quoting can protect it.

Locations:

- `action.yml:148`
- `action.yml:118`
- `action.yml:185`
- `action.yml:205`

### script-injection (severity: high)

Sub-rule (a): Direct expression interpolation in run: block in autotag-releases.yml. The 'Create and push tags' step interpolates `${{ steps.tag_name.outputs.current_version }}` directly into shell commands: `MINOR="$(echo -n ${{ steps.tag_name.outputs.current_version }} | cut -d. -f1-2)"` and `MAJOR="$(echo -n ${{ steps.tag_name.outputs.current_version }} | cut -d. -f1)"`. The step output current_version is derived from GITHUB_REF and could be manipulated via a crafted tag name to inject shell metacharacters.

Locations:

- `.github/workflows/autotag-releases.yml:22`

### github-env-injection (severity: high)

Unsanitized user-controlled inputs are written to $GITHUB_ENV without the required `printf '%s' ... | tr -d '\n\r'` sanitization step:
1. `_srt_NEW_RUSTFLAGS` (set from `inputs.rustflags`) is written as `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` — an attacker-supplied rustflags value containing newlines could inject arbitrary environment variables.
2. `_srt_CARGO_BUILD_WARNINGS` (set from `inputs.build-warnings`) is written as `echo "CARGO_BUILD_WARNINGS=$_srt_CARGO_BUILD_WARNINGS" >> $GITHUB_ENV` — same risk.
3. In the 'flags' step, `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` writes an expression derived from inputs.toolchain and inputs.components directly to $GITHUB_OUTPUT without sanitization.

Locations:

- `action.yml:130`
- `action.yml:133`
- `action.yml:118`

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. This pattern executes whatever the remote server returns without first downloading and verifying the script, making it vulnerable to MITM attacks or server-side compromise.

Locations:

- `action.yml:153`

### unpinned-uses (severity: high)

The following `uses:` references use mutable version tags instead of pinned 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved:
- `actions/checkout@v7.0.1` in ci.yml (appears twice, in the 'install' and 'cache' jobs)
- `actions/checkout@v7.0.1` in autotag-releases.yml
These should be pinned to a full SHA, e.g. `actions/checkout@<40-char-sha> # v7.0.1`.

Locations:

- `.github/workflows/ci.yml:31`
- `.github/workflows/ci.yml:68`
- `.github/workflows/autotag-releases.yml:14`

### missing-permissions (severity: medium)

The workflow file ci.yml has no top-level `permissions:` key and none of its jobs (install, cache) define a `permissions:` block. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (e.g., write access to contents, packages, etc.). A top-level `permissions:` block with minimal scopes (or per-job permissions) should be added.

Locations:

- `.github/workflows/ci.yml:1`

### broad-permissions (severity: medium)

The workflow file autotag-releases.yml sets `permissions: read-all` at the top level. This grants read access to all repository scopes, which is broader than necessary. The top-level permissions should be set to the minimal required scopes (e.g., `permissions: {}` or specific scopes only), relying on the job-level `permissions: contents: write` for the job that needs it.

Locations:

- `.github/workflows/autotag-releases.yml:8`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell, unpinned-uses, missing-permissions, broad-permissions

**Notes:**

Fixed all 7 findings across action.yml, ci.yml, and autotag-releases.yml:

1. script-injection (action.yml, 4 locations): Moved all ${{ }} expressions out of run: blocks into env: blocks — github.action_path, steps.flags.outputs.*, steps.versions.outputs.rustc-version, and the contains() expression for downgrade.

2. script-injection (autotag-releases.yml): Moved steps.tag_name.outputs.current_version into env block as CURRENT_VERSION, replaced 'echo -n ${{ ... }}' with 'printf "%s" "$CURRENT_VERSION"'.

3. github-env-injection (action.yml): Added printf '%s' | tr -d '\n\r' sanitization before writing RUSTFLAGS and CARGO_BUILD_WARNINGS to $GITHUB_ENV, and before writing downgrade to $GITHUB_OUTPUT.

4. unsafe-shell (action.yml): Replaced 'curl ... | sh -s -- args' with download-to-tempfile then execute pattern. Dropped the '--' separator (it was the shell's option terminator, not the script's).

5. unpinned-uses: Pinned all 3 occurrences of actions/checkout@v7.0.1 to full SHA 3d3c42e5aac5ba805825da76410c181273ba90b1.

6. missing-permissions (ci.yml): Added top-level 'permissions: {}' block.

7. broad-permissions (autotag-releases.yml): Replaced top-level 'permissions: read-all' with 'permissions: {}', keeping the job-level 'permissions: contents: write' for the job that needs it.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed all three findings in action.yml:

1. github-env-injection (flags step, lines 110-111): Added `printf '%s' ... | tr -d '\n\r'` sanitization for both `_srt_targets` and `_srt_components` before writing to $GITHUB_OUTPUT.

2. script-injection in flags step (lines 110-111): Replaced unquoted `${_srt_targets//,/ }` and `${_srt_components//,/ }` expansions inside command substitution with safe `IFS=',' read -ra` array splitting and quoted `"${arr[@]}"` iteration to build the flags strings.

3. script-injection in rustup toolchain install step (lines 175, 178, 193, 200):
   - `rustup component add`: now uses `IFS=',' read -ra` array with `"${_srt_comp_args[@]}"`
   - `rustup target add`: now uses `IFS=',' read -ra` array with `"${_srt_tgt_args[@]}"`
   - `rustup toolchain install`: now uses `IFS=',' read -ra` array for toolchain args; `--profile minimal${_srt_flags_downgrade}` replaced with a `_srt_profile_args` array that conditionally appends `--allow-downgrade`
   - `rustup override set`: now uses `"${_srt_toolchain_args[-1]}"` (last element of already-built array) instead of unquoted `${_srt_toolchain//*,/ }`

### Iteration 3

**Fixes applied:** script-injection, github-env-injection

**Notes:**

1. Fixed script-injection in .github/workflows/ci.yml (lines 60, 63, 66): Moved `${{steps.toolchain.outputs.rustc-version}}`, `${{steps.toolchain.outputs.cargo-version}}`, and `${{steps.toolchain.outputs.rustup-version}}` expressions out of `run:` blocks into `env:` blocks as RUSTC_VERSION, CARGO_VERSION, and RUSTUP_VERSION respectively. The shell commands now reference plain environment variables.
2. Fixed github-env-injection in action.yml (lines 155, 157): The 'Install rustup, if needed' step now sanitizes CARGO_HOME/USERPROFILE/HOME-derived paths before writing to $GITHUB_PATH. Each path is captured via `printf '%s' ... | tr -d '\n\r'` (plus `sed` for Windows backslash conversion) into a local variable `_srt_cargo_bin`, then written with `printf '%s\n'` to prevent newline injection.

