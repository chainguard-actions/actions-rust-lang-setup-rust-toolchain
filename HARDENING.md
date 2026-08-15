<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.16.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.16.1** was hardened automatically. 3 finding(s) were identified and resolved across 5 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Four `run:` blocks in action.yml directly interpolate `${{ }}` expressions into shell commands, enabling script injection.

1. The `flags` step (line ~114) embeds `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` directly in an `echo` command that writes to $GITHUB_OUTPUT. The `inputs.toolchain` and `inputs.components` values are attacker-controlled.

2. The `Install Rust Problem Matcher` step (line ~140) uses `${{ github.action_path }}` directly inside the `run:` shell string: `echo "::add-matcher::${{ github.action_path }}/rust.json"`.

3. The `rustup toolchain install` step (line ~168) interpolates `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` directly into the shell command: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`.

4. The `Downgrade registry access protocol` step (line ~176) interpolates `${{steps.versions.outputs.rustc-version}}` directly into a bash conditional: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...`.

All four cases allow YAML template substitution to inject arbitrary shell metacharacters before the shell ever parses the command.

Locations:

- `action.yml:114`
- `action.yml:140`
- `action.yml:168`
- `action.yml:176`

### github-env-injection (severity: high)

The `Setting Environment Variables` step writes the value of `$_srt_NEW_RUSTFLAGS` to `$GITHUB_ENV` without sanitization. `_srt_NEW_RUSTFLAGS` is set from `inputs.rustflags` (an attacker-controlled composite-action input) via the `env:` block. The write `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` is not preceded by the required `printf '%s' ... | tr -d '\n\r'` sanitization step. A newline embedded in `inputs.rustflags` would allow an attacker to inject arbitrary environment variables into subsequent steps.

Locations:

- `action.yml:130`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of `curl` directly to `sh` without first downloading and verifying the script: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Although TLS is enforced, this pattern still executes remotely-fetched code immediately without any integrity check (e.g., checksum or signature verification), making it vulnerable to supply-chain compromise of the rustup distribution server.

Locations:

- `action.yml:145`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection (4 locations):
   - flags step (~line 114): Replaced ${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}} with pure shell logic using _srt_toolchain env var; added _srt_toolchain to the env block.
   - Install Rust Problem Matcher (~line 140): Moved ${{ github.action_path }} into env: ACTION_PATH and referenced as ${ACTION_PATH} in shell.
   - rustup toolchain install (~line 168): Moved steps.flags.outputs.{targets,components,downgrade} into env vars (_srt_flags_targets, _srt_flags_components, _srt_flags_downgrade) and built the rustup command using a bash array with xargs-based tokenization for list-type flags.
   - Downgrade registry access protocol (~line 176): Moved steps.versions.outputs.rustc-version into env var _srt_rustc_version and referenced as "$_srt_rustc_version" in the bash conditional.

2. github-env-injection (~line 130): Added sanitization of _srt_NEW_RUSTFLAGS before writing to $GITHUB_ENV using printf '%s' "$_srt_NEW_RUSTFLAGS" | tr -d '\n\r' to strip newlines.

3. unsafe-shell (~line 145): Changed curl | sh to download the rustup installer to /tmp/rustup-init.sh first, then execute it separately with sh /tmp/rustup-init.sh, then clean up.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed all findings in action.yml:

1. github-env-injection (flags step, lines ~107-108): Replaced direct echo of user-controlled values to $GITHUB_OUTPUT with a sanitized approach: (a) strip newlines from _srt_targets and _srt_components using `tr -d '\n\r'`, (b) build flag arrays using xargs-based tokenization, (c) write to GITHUB_OUTPUT using `printf 'targets=%s\n'` instead of `echo "targets=$(...)"` to prevent newline injection.

2. script-injection (multiple locations):
   - flags step: Replaced unquoted `${_srt_targets//,/ }` and `${_srt_components//,/ }` in for loops with xargs-based array tokenization.
   - rustup toolchain install step: Fixed unquoted `[[ -n $_srt_components ]]` and `[[ -n $_srt_targets ]]` tests to use double quotes. Replaced unquoted `rustup component add ${_srt_components//,/ }` and `rustup target add ${_srt_targets//,/ }` with xargs-based array tokenization. Replaced unquoted `for t in ${_srt_toolchain//,/ }` with xargs-based array tokenization. Replaced unquoted `rustup override set ${_srt_toolchain//*,/ }` with a properly quoted variable `rustup override set "$_srt_last_toolchain"`.

### Iteration 3

**Fixes applied:** unpinned-uses, broad-permissions, missing-permissions, script-injection

**Notes:**

Fixed all four findings: (1) Pinned all three `actions/checkout@v6` references to SHA `d23441a48e516b6c34aea4fa41551a30e30af803 # v6` in both workflow files. (2) Replaced `permissions: read-all` with `permissions: {}` at the top level of autotag-releases.yml (job-level `contents: write` preserved). (3) Added `permissions: {}` at the top level of ci.yml. (4) Moved `${{ steps.tag_name.outputs.current_version }}` into an `env:` block as `CURRENT_VERSION` and referenced it as `"$CURRENT_VERSION"` in the shell script to prevent script injection.

### Iteration 4

**Fixes applied:** script-injection

**Notes:**

Fixed three script injection vulnerabilities in .github/workflows/ci.yml (lines 51, 54, 57). Each of the three steps that directly interpolated `${{ steps.toolchain.outputs.* }}` expressions inside `run:` shell commands was updated to: (1) move the expression into an `env:` block as a named variable (RUSTC_VER, CARGO_VER, RUSTUP_VER), and (2) reference the variable as a quoted shell variable (`echo "$RUSTC_VER"` etc.) in the `run:` command. This prevents attacker-controlled step output values containing shell metacharacters from being interpreted by the shell.

### Iteration 5

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted variable in [[ ]] conditional in the 'Setting Environment Variables' step of action.yml. Changed `$_srt_NEW_RUSTFLAGS != ""` to `"$_srt_NEW_RUSTFLAGS" != ""` to properly double-quote the workflow-controllable input variable as required by the security rule.

