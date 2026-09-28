<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.17.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.17.0** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings (sub-rule a), allowing template substitution before the shell ever sees the value:

1. `flags` step (line ~117): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — inputs.toolchain and inputs.components are interpolated directly.

2. `Install Rust Problem Matcher` step (line ~140): `echo "::add-matcher::${{ github.action_path }}/rust.json"` — github.action_path interpolated directly in run.

3. `rustup toolchain install` step (line ~170): `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — steps.flags.outputs.* expressions interpolated directly into a shell command.

4. `Downgrade registry access protocol` step (line ~192): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...` — steps.versions.outputs.rustc-version interpolated directly into a shell conditional.

Locations:

- `action.yml:117`
- `action.yml:140`
- `action.yml:170`
- `action.yml:192`

### github-env-injection (severity: high)

Untrusted input values are written to GitHub special environment files without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`):

1. `flags` step (lines ~115-116): `inputs.target` and `inputs.components` are placed into env vars `_srt_targets` and `_srt_components`, then written to `$GITHUB_OUTPUT` via `echo "targets=$(for t in ${_srt_targets//,/ }; ...)" >> $GITHUB_OUTPUT` and `echo "components=$(for c in ${_srt_components//,/ }; ...)" >> $GITHUB_OUTPUT` — no newline sanitization applied before the write.

2. `Setting Environment Variables` step (line ~131): `inputs.rustflags` is placed into env var `_srt_NEW_RUSTFLAGS`, then written to `$GITHUB_ENV` via `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` — no newline sanitization applied before the write. An attacker-controlled `rustflags` input containing a newline could inject arbitrary environment variables.

Locations:

- `action.yml:115`
- `action.yml:116`
- `action.yml:131`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes remote content directly to a shell interpreter without first downloading and verifying it: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote server or network is compromised, arbitrary code executes on the runner.

Locations:

- `action.yml:144`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all findings in hardened/action/action.yml:

1. script-injection (lines 117, 140, 170, 192): Moved all ${{ }} expressions out of run: shell strings into step env: blocks. The flags step's downgrade expression is now computed as a boolean env var and the shell uses a conditional. The Install Rust Problem Matcher step uses _srt_action_path env var. The rustup toolchain install step uses _srt_flags_targets, _srt_flags_components, _srt_flags_downgrade env vars. The Downgrade registry step uses _srt_rustc_version env var.

2. github-env-injection (lines 115, 116, 131): Added newline sanitization using `printf '%s' ... | tr -d '\n\r'` before writing targets and components to $GITHUB_OUTPUT, and before writing RUSTFLAGS to $GITHUB_ENV.

3. unsafe-shell (line 144): Replaced `curl ... | sh -s -- --default-toolchain none -y` with downloading to a mktemp file then executing `sh "$_srt_rustup_init" --default-toolchain none -y` (dropping the shell's -s and -- options as required), followed by cleanup with rm -f.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all unquoted variable expansions in the 'rustup toolchain install' step's run block:
1. `_srt_components`: Added double-quotes to the `-n` test and used `IFS=',' read -ra` to split into an array, then `"${_srt_components_arr[@]}"` for safe expansion.
2. `_srt_targets`: Same approach with `IFS=',' read -ra _srt_targets_arr`.
3. `_srt_toolchain`: Split with `IFS=',' read -ra _srt_toolchain_arr`; last element accessed via `"${_srt_toolchain_arr[-1]}"` instead of unquoted `${_srt_toolchain//*,/ }`.
4. `_srt_flags_targets`, `_srt_flags_components`, `_srt_flags_downgrade`: These are space-separated flag tokens (e.g., `--target x86_64-unknown-linux-gnu`); tokenized using the xargs NUL-delimited loop pattern into arrays and expanded with `"${arr[@]}"`.
All expansions are now properly double-quoted, preventing shell metacharacter injection via attacker-controlled inputs.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed the script-injection finding in the `flags` step (lines 111-112 of action.yml). The fix restructures the for loops to: (1) use `set -f` before the loops to disable glob expansion, preventing glob characters (*, ?, [) in input values from being expanded; (2) build the output strings in separate variables with loop variables `t` and `c` inside double-quoted string assignments rather than unquoted in echo arguments; (3) use `set +f` after the loops to restore glob expansion. The output is then written with properly quoted variables. This addresses both sub-issues: unquoted for-loop iterators (now protected by set -f) and unquoted loop variables in echo (now inside double-quoted string assignments).

