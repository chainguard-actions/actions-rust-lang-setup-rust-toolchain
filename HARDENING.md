<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.17.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.17.0** was hardened automatically. 3 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple `run:` blocks in action.yml directly interpolate `${{ ... }}` expressions into shell command strings, enabling script injection.

1. (line ~115) The `flags` step writes a `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` expression directly into an `echo` command: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`

2. (line ~138) The `Install Rust Problem Matcher` step interpolates `${{ github.action_path }}` directly in a `run:` shell string: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"`

3. (line ~163) The `rustup toolchain install` step interpolates `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` directly into a shell command: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`

4. (line ~183) The `Downgrade registry access protocol` step interpolates `${{steps.versions.outputs.rustc-version}}` directly into a bash conditional: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...`

All four cases allow YAML template substitution to inject arbitrary shell metacharacters before the shell ever parses the string.

Locations:

- `action.yml:115`
- `action.yml:138`
- `action.yml:163`
- `action.yml:183`

### github-env-injection (severity: high)

Multiple `run:` steps write values derived from untrusted inputs to GitHub special environment files without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

1. (line ~115) The `flags` step writes a value derived directly from `inputs.toolchain` and `inputs.components` (via `${{contains(inputs.toolchain, 'nightly') && inputs.components && ...}}`) to `$GITHUB_OUTPUT` with no sanitization: `echo "downgrade=${{...}}" >> $GITHUB_OUTPUT`

2. (line ~130) The `Setting Environment Variables` step writes `$_srt_NEW_RUSTFLAGS` — sourced from `inputs.rustflags` via the env var `_srt_NEW_RUSTFLAGS: ${{inputs.rustflags}}` — to `$GITHUB_ENV` without sanitization: `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV`. A caller-controlled `inputs.rustflags` value containing newlines could inject arbitrary environment variables.

3. (line ~186) The `Downgrade registry access protocol` step conditionally writes to `$GITHUB_ENV` inside a branch guarded by `${{steps.versions.outputs.rustc-version}}` interpolated directly into the shell conditional — the guard itself is subject to injection (see script-injection finding), and the write is unsanitized: `echo "CARGO_REGISTRIES_CRATES_IO_PROTOCOL=git" >> $GITHUB_ENV`

Locations:

- `action.yml:115`
- `action.yml:130`
- `action.yml:186`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of a remote `curl` request directly to `sh` for execution without first saving it to a file and verifying its integrity. Pattern: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Although TLS is enforced, piping remote content directly to a shell interpreter is an unsafe pattern — a compromised or MitM'd server could deliver malicious code that executes immediately.

Locations:

- `action.yml:143`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all findings in action.yml:

1. script-injection (4 locations): Moved all ${{ }} expressions out of run: shell strings into env: blocks. Specifically: (a) `contains(inputs.toolchain, 'nightly') && inputs.components && ...` → `_srt_downgrade` env var; (b) `github.action_path` → `_srt_action_path` env var; (c) `steps.flags.outputs.targets/components/downgrade` → `_srt_flags_targets/_srt_flags_components/_srt_flags_downgrade` env vars; (d) `steps.versions.outputs.rustc-version` → `_srt_rustc_version` env var.

2. github-env-injection (2 active locations): (a) `_srt_downgrade` sanitized with `printf '%s' "$_srt_downgrade" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT; (b) `_srt_NEW_RUSTFLAGS` sanitized with `printf '%s' "$_srt_NEW_RUSTFLAGS" | tr -d '\n\r'` before writing to $GITHUB_ENV. The third location (line ~186) writes a hardcoded literal 'git' which is not user-controlled.

3. unsafe-shell (line ~143): Replaced `curl ... | sh -s -- --default-toolchain none -y` with: download to temp file via `mktemp`, execute with `sh "$_srt_rustup_init" --default-toolchain none -y` (dropping `--` as required since we're no longer using `sh -s`), then clean up with `rm -f`.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed two security findings in hardened/action/action.yml:

1. github-env-injection (lines 113-114): The `flags` step now captures for-loop output into raw variables first, then sanitizes with `printf '%s' ... | tr -d '\n\r'` before writing `targets` and `components` to $GITHUB_OUTPUT. Previously these were written directly without sanitization.

2. script-injection (multiple locations):
   - flags step: Changed `echo -n ' --target' $t` to `printf ' --target %s' "$t"` to properly quote the loop variable.
   - Setting Environment Variables step: Added quotes around `"$_srt_NEW_RUSTFLAGS"` in the [[ ]] conditional.
   - rustup toolchain install step: Replaced all unquoted variable expansions with properly quoted array expansions. Used `IFS=',' read -ra` to split comma-separated inputs (_srt_components, _srt_targets, _srt_toolchain) into bash arrays. Used xargs-based tokenization (while/read -d '' loop) for space-separated flag strings from prior step outputs (_srt_flags_targets, _srt_flags_components, _srt_flags_downgrade). Used `"${_srt_toolchain_arr[-1]}"` for the last toolchain element instead of the unquoted `${_srt_toolchain//*,/ }` pattern.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed the script injection vulnerability in the `flags` step of action.yml (lines 116 and 119). Replaced unquoted for-loop expansions `for t in ${_srt_targets//,/ }` and `for c in ${_srt_components//,/ }` with safe IFS-based array splitting using `IFS=',' read -ra _srt_t_arr <<< "$_srt_targets"` and `IFS=',' read -ra _srt_c_arr <<< "$_srt_components"`, followed by quoted array expansion `for t in "${_srt_t_arr[@]}"` and `for c in "${_srt_c_arr[@]}"`. This prevents glob characters (*, ?, [) and unexpected word-splitting from attacker-controlled inputs from being evaluated by the shell.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed all 5 occurrences of IFS=',' read -ra splitting that only split on commas but upstream split on commas and whitespace. Replaced each with the xargs idiom: `printf '%s' "${VAR//,/ }" | xargs printf '%s\0'` read via a null-delimited while loop. Added `if [[ -n "$VAR" ]]; then ... fi` guards where missing to prevent empty-input issues. Locations fixed: (1) _srt_t_arr in flags step, (2) _srt_c_arr in flags step, (3) _srt_comp_arr in toolchain install step, (4) _srt_tgt_arr in toolchain install step, (5) _srt_toolchain_arr in toolchain install step.

