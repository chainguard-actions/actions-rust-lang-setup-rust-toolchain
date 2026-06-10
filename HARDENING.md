# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.16.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `ff50f15e4b79bfbf764dafdfd2579175a6ea9771`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **actions-rust-lang--setup-rust-toolchain/v1.16.1** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

The 'Install Rust Problem Matcher' run: block directly interpolates ${{ github.action_path }} into the shell command string without first assigning it to an environment variable. Any github.* expression interpolated directly in a run: block is a script-injection risk.

Locations:

- `action.yml:116`

### script-injection (severity: high)

The 'flags' step run: block directly interpolates ${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}} into the shell command string (echoed to $GITHUB_OUTPUT). The inputs.toolchain and inputs.components values are attacker-controlled and are interpolated directly into the run: shell without being routed through env: variables first.

Locations:

- `action.yml:101`

### github-env-injection (severity: high)

The 'Setting Environment Variables' step writes the attacker-controlled inputs.rustflags value (via env var _srt_NEW_RUSTFLAGS) to $GITHUB_ENV with `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` without applying the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A malicious value containing newlines could inject arbitrary environment variables into subsequent steps.

Locations:

- `action.yml:110`

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. The script is not downloaded to a file first and verified before execution.

Locations:

- `action.yml:126`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all four findings in action.yml:
1. script-injection (flags step, line 101): Moved contains() expression into env: block as _srt_downgrade, referenced as shell variable.
2. github-env-injection (Setting Environment Variables, line 110): Added printf/tr sanitization to strip newlines from _srt_NEW_RUSTFLAGS before writing to $GITHUB_ENV.
3. script-injection (Install Rust Problem Matcher, line 116): Moved ${{ github.action_path }} into env: block as _srt_ACTION_PATH, referenced as shell variable.
4. unsafe-shell (Install rustup, line 126): Replaced curl|sh pipe with download-to-tempfile, chmod, execute, cleanup pattern.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed the `flags` step in action.yml (lines 110-112) to sanitize attacker-controlled input values before writing to $GITHUB_OUTPUT. Added intermediate variables (_srt_targets_safe, _srt_components_safe, _srt_downgrade_safe) that use `printf '%s' ... | tr -d '\n\r'` to strip newlines from the computed values before the echo statements write them to $GITHUB_OUTPUT. This prevents newline injection attacks that could inject arbitrary key=value pairs into the GitHub output context. The fix follows the same sanitization pattern already used in the `Setting Environment Variables` step for RUSTFLAGS.

