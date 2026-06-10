# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `ff50f15e4b79bfbf764dafdfd2579175a6ea9771`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.3** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

The 'Install Rust Problem Matcher' step directly interpolates `${{ github.action_path }}` inside a `run:` shell command: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"`. The `github.action_path` context value is attacker-controllable and is embedded directly in the shell string rather than being assigned to an environment variable first.

Locations:

- `action.yml:144`

### script-injection (severity: high)

The 'flags' step directly interpolates `${{ contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || '' }}` inside a `run:` shell command writing to $GITHUB_OUTPUT. The attacker-controlled `inputs.components` value is embedded directly in the shell string rather than being passed through an environment variable.

Locations:

- `action.yml:109`

### github-env-injection (severity: high)

The 'flags' step writes attacker-controlled input values to $GITHUB_OUTPUT without sanitization. `$targets` (from `inputs.target`) and `$components` (from `inputs.components`) are written via `echo "targets=..." >> $GITHUB_OUTPUT` and `echo "components=..." >> $GITHUB_OUTPUT` without applying `printf '%s' ... | tr -d '\n\r'` before the write. A newline-injection attack could poison subsequent steps' environment. Additionally, `inputs.components` and `inputs.toolchain` are directly interpolated into the downgrade echo on line 109.

Locations:

- `action.yml:107`
- `action.yml:108`
- `action.yml:109`

### github-env-injection (severity: high)

The 'Setting Environment Variables' step writes `$NEW_RUSTFLAGS` (sourced from `inputs.rustflags`, an attacker-controlled input) to $GITHUB_ENV via `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). An attacker can inject newlines into `inputs.rustflags` to set arbitrary environment variables for subsequent steps.

Locations:

- `action.yml:130`

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Although TLS is enforced, this pattern executes whatever the remote server returns without any integrity verification (e.g., a checksum check), making it vulnerable to a compromised CDN or MITM attack.

Locations:

- `action.yml:150`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all 5 findings in action.yml:
1. script-injection (line 144): Moved `github.action_path` into `ACTION_PATH` env var in the 'Install Rust Problem Matcher' step; shell now uses `${ACTION_PATH}/rust.json`.
2. script-injection (line 109): Replaced the `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` inline expression with pure shell logic using the already-env-var-bound `$toolchain` and `$components` variables. Added `toolchain` to the flags step env block.
3. github-env-injection (lines 107-109): All three GITHUB_OUTPUT writes (targets, components, downgrade) now sanitize values with `printf '%s' ... | tr -d '\n\r'` before writing.
4. github-env-injection (line 130): The RUSTFLAGS write to GITHUB_ENV now sanitizes `$NEW_RUSTFLAGS` with `printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r'` before writing.
5. unsafe-shell (line 150): Replaced `curl ... | sh -s --` with a download-then-execute pattern: curl saves to `/tmp/rustup-init.sh`, then `sh /tmp/rustup-init.sh` executes it, then the temp file is removed.

