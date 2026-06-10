# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `ff50f15e4b79bfbf764dafdfd2579175a6ea9771`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.4** was hardened automatically. 5 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

The 'flags' step directly interpolates `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` inside a `run:` shell command (writing to $GITHUB_OUTPUT) rather than routing through an env: variable. Attacker-controlled `inputs.*` values are interpolated directly into the shell command string.

Locations:

- `action.yml:103`

### script-injection (severity: high)

The 'Install Rust Problem Matcher' step directly interpolates `${{ github.action_path }}` inside a `run:` shell command: `echo "::add-matcher::${{ github.action_path }}/rust.json"`. The `github.*` context is interpolated directly into the shell string rather than being assigned to an env: variable first.

Locations:

- `action.yml:131`

### github-env-injection (severity: high)

The 'flags' step writes an attacker-controlled `inputs.*` expression directly into $GITHUB_OUTPUT without sanitization: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`. A newline injected via inputs.toolchain or inputs.components could poison the output file.

Locations:

- `action.yml:103`

### github-env-injection (severity: high)

The 'Setting Environment Variables' step routes `inputs.rustflags` through the env var `NEW_RUSTFLAGS` and then writes it to $GITHUB_ENV with `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV` without the required sanitization step (`printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r'`). A newline injected via the `rustflags` input could allow an attacker to set arbitrary environment variables.

Locations:

- `action.yml:120`

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step downloads and executes a remote script by piping curl directly to sh: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote server or network is compromised, arbitrary code could be executed on the runner.

Locations:

- `action.yml:136`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all 5 findings in action.yml:
1. (script-injection, line 103) Moved `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` out of the run: shell string into an env: variable `DOWNGRADE_FLAG`.
2. (github-env-injection, line 103) Added `printf '%s' "$DOWNGRADE_FLAG" | tr -d '\n\r'` sanitization before writing the downgrade flag to $GITHUB_OUTPUT.
3. (github-env-injection, line 120) Added `printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r'` sanitization before writing RUSTFLAGS to $GITHUB_ENV.
4. (script-injection, line 131) Moved `${{ github.action_path }}` out of the run: shell string into an env: variable `ACTION_PATH`, referenced as `${ACTION_PATH}` in the shell.
5. (unsafe-shell, line 136) Replaced `curl ... | sh` with: download script to `rustup-init.sh`, execute it with `sh rustup-init.sh`, then `rm -f rustup-init.sh`.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed the 'flags' step in action.yml to sanitize `targets` and `components` before writing to $GITHUB_OUTPUT. Both values are now captured into intermediate variables (raw_targets, raw_components), then passed through `printf '%s' ... | tr -d '\n\r'` to strip embedded newlines before the echo to $GITHUB_OUTPUT. This matches the existing pattern used for DOWNGRADE_FLAG in the same step.

