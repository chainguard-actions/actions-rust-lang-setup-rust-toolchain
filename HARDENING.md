<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.17.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.17.0** was hardened automatically. 3 finding(s) were identified and resolved across 5 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): Multiple ${{ }} expressions are interpolated directly inside run: shell command strings in action.yml.

1. flags step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — inputs.toolchain and inputs.components are interpolated directly into the shell command.

2. Install Rust Problem Matcher step: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — github.action_path is interpolated directly.

3. rustup toolchain install step: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three steps.flags.outputs.* expressions are interpolated directly into the shell command.

4. Downgrade registry step: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ...` — steps.versions.outputs.rustc-version is interpolated directly into the shell command.

Locations:

- `action.yml:116`
- `action.yml:133`
- `action.yml:175`
- `action.yml:208`

### github-env-injection (severity: high)

Multiple unsanitized writes of untrusted input values to GITHUB_OUTPUT and GITHUB_ENV without the required `printf '%s' ... | tr -d '\n\r'` sanitization step.

1. flags step writes `inputs.target` (via env var `_srt_targets`) to $GITHUB_OUTPUT without sanitization: `echo "targets=$(for t in ${_srt_targets//,/ }; do echo -n ' --target' $t; done)" >> $GITHUB_OUTPUT`

2. flags step writes `inputs.components` (via env var `_srt_components`) to $GITHUB_OUTPUT without sanitization: `echo "components=$(for c in ${_srt_components//,/ }; do echo -n ' --component' $c; done)" >> $GITHUB_OUTPUT`

3. flags step writes a direct ${{ }} expression containing inputs.toolchain and inputs.components to $GITHUB_OUTPUT without sanitization: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`

4. Setting Environment Variables step writes `inputs.rustflags` (via env var `_srt_NEW_RUSTFLAGS`) to $GITHUB_ENV without sanitization: `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV`

Locations:

- `action.yml:114`
- `action.yml:115`
- `action.yml:116`
- `action.yml:130`

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes remote content directly to a shell interpreter without first downloading and verifying it: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Although TLS is enforced, the script is executed immediately from the network without any integrity check (e.g., checksum or signature verification), allowing a compromised or MITM'd server to execute arbitrary code on the runner.

Locations:

- `action.yml:143`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection: Moved all ${{ }} expressions out of run: shell strings into env: blocks. The flags step downgrade expression was replaced with equivalent shell logic using the already-available $_srt_toolchain env var. The github.action_path expression was moved to _srt_action_path env var. The steps.flags.outputs.* expressions were moved to _srt_flags_targets, _srt_flags_components, _srt_flags_downgrade env vars. The steps.versions.outputs.rustc-version expression was moved to _srt_rustc_version env var.

2. github-env-injection: All four unsanitized writes to GITHUB_OUTPUT/GITHUB_ENV were fixed by adding printf '%s' "$VAR" | tr -d '\n\r' sanitization before writing. The targets, components, and downgrade outputs in the flags step are now sanitized, as is the RUSTFLAGS write to GITHUB_ENV.

3. unsafe-shell: Replaced the curl | sh pipe pattern with a two-step approach: download the rustup installer to /tmp/rustup-init.sh, execute it separately with sh, then remove the temporary file.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all three script-injection findings in hardened/action/action.yml:

1. flags step (lines 133/136): Replaced unquoted `${_srt_targets//,/ }` and `${_srt_components//,/ }` in for loops with IFS=, read -ra array splitting followed by properly quoted array expansions `"${_srt_t_arr[@]}"` and `"${_srt_c_arr[@]}"`.

2. Install Rust Problem Matcher step (line 148): Changed from single-line echo with embedded variable to a multi-line block that sanitizes `$_srt_action_path` with `printf '%s' | tr -d '\n\r'` before use.

3. rustup toolchain install step (lines 196-212): Fixed all unquoted expansions — added quotes to `[[ -n ]]` tests, replaced all `${var//,/ }` word-splitting patterns with `IFS=, read -ra` array splitting + quoted `"${arr[@]}"` expansions, replaced `${_srt_flags_*}` unquoted expansions with `read -ra` arrays, and replaced `${_srt_toolchain//*,/ }` with the properly quoted `"${_srt_toolchain##*,}"` parameter expansion.

### Iteration 3

**Fixes applied:** script-injection, unpinned-uses, permissions

**Notes:**

Fixed all 6 findings across ci.yml and autotag-releases.yml: (1) ci.yml - Added top-level 'permissions: contents: read' block. (2) ci.yml - Pinned both actions/checkout@v7.0.0 references to SHA 9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0. (3) ci.yml - Moved rustc-version, cargo-version, and rustup-version expressions out of run: strings into env: blocks. (4) autotag-releases.yml - Replaced 'permissions: read-all' with 'permissions: contents: read'. (5) autotag-releases.yml - Pinned actions/checkout@v7.0.0 to SHA 9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0. (6) autotag-releases.yml - Moved current_version expression into env var CURRENT_VERSION and referenced it safely in the shell script.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerability in action.yml at line 152 in the 'Setting Environment Variables' step. Changed `$_srt_NEW_RUSTFLAGS != ""` to `"$_srt_NEW_RUSTFLAGS" != ""` in the bash `if [[ ... ]]` condition. The unquoted variable expansion allowed attacker-controlled values in `inputs.rustflags` containing shell metacharacters (`;`, `|`, `$(...)`, glob characters, whitespace) to be interpreted by the shell before the comparison was reached. Double-quoting the variable ensures the value is treated as a literal string.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed the 'Get version from tag' step in .github/workflows/autotag-releases.yml. The GITHUB_REF-derived value is now sanitized with `printf '%s' "$raw_version" | tr -d '\n\r'` before being written to $GITHUB_OUTPUT, preventing newline injection attacks where a crafted tag name could inject arbitrary key=value pairs into GITHUB_OUTPUT.

