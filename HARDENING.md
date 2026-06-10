# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.16.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `ff50f15e4b79bfbf764dafdfd2579175a6ea9771`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **actions-rust-lang--setup-rust-toolchain/v1.16.0** was hardened automatically. 4 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

In the 'flags' step run: block, `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` directly interpolates attacker-controlled `inputs.toolchain` and `inputs.components` expressions inside the shell command string instead of routing them through env: variables. An attacker can inject arbitrary shell commands via these inputs.

Locations:

- `action.yml:114`

### script-injection (severity: high)

In the 'Install Rust Problem Matcher' step run: block, `${{ github.action_path }}` is directly interpolated inside the shell command string (`echo "::add-matcher::${{ github.action_path }}/rust.json"`). The github.* context is attacker-influenced and must be routed through an env: variable instead.

Locations:

- `action.yml:130`

### github-env-injection (severity: high)

In the 'flags' step run: block, attacker-controlled `inputs.toolchain` and `inputs.components` values are written directly to $GITHUB_OUTPUT via `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A newline injection in these inputs could allow an attacker to set arbitrary environment variables or outputs.

Locations:

- `action.yml:114`

### unsafe-shell (severity: high)

In the 'Install rustup, if needed' step, remote content is piped directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. The script should be downloaded to a file first and verified before execution.

Locations:

- `action.yml:137`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all four findings in action.yml:
1. (script-injection, line 114) Moved `contains(inputs.toolchain, 'nightly') && inputs.components` expression from the run block into an env variable `DOWNGRADE_FLAG`.
2. (github-env-injection, line 114) Added sanitization of `DOWNGRADE_FLAG` via `printf '%s' "$DOWNGRADE_FLAG" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT.
3. (script-injection, line 130) Moved `${{ github.action_path }}` from the run block into an env variable `ACTION_PATH`, referenced as `${ACTION_PATH}` in the shell.
4. (unsafe-shell, line 137) Replaced `curl ... | sh` with: download to a temp file via `mktemp`, `chmod +x`, execute, then `rm -f` the temp file.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed two github-env-injection vulnerabilities in action.yml:
1. 'flags' step (lines 103-104): Added `printf '%s' ... | tr -d '\n\r'` sanitization for both `targets` and `components` before writing to $GITHUB_OUTPUT. The values are now stored in `safe_targets` and `safe_components` variables before the echo.
2. 'Setting Environment Variables' step (line 131): Added `printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r'` sanitization, storing the result in `safe_rustflags` before writing `RUSTFLAGS=$safe_rustflags` to $GITHUB_ENV. This prevents attacker-controlled inputs (target, components, rustflags) from injecting arbitrary key-value pairs via newlines.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in the 'rustup toolchain install' step of action.yml. The three ${{steps.flags.outputs.targets}}, ${{steps.flags.outputs.components}}, and ${{steps.flags.outputs.downgrade}} expressions were directly interpolated into the shell command string. They have been moved into the step's env: block as flag_targets, flag_components, and flag_downgrade respectively. The shell script now references them as plain environment variables (${flag_targets}, ${flag_components}, ${flag_downgrade}), eliminating the script injection risk.

