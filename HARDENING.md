# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `ff50f15e4b79bfbf764dafdfd2579175a6ea9771`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.2** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. The script is not downloaded to a file first and inspected before execution, making this an unsafe shell pattern.

Locations:

- `action.yml:131`

### script-injection (severity: high)

GitHub Actions expressions are interpolated directly into run: shell command strings without being assigned to environment variables first.

(1) In the 'Install Rust Problem Matcher' step: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — the `github.action_path` context value is embedded directly in the shell command.

(2) In the 'flags' step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the shell command string.

Locations:

- `action.yml:127`
- `action.yml:104`

### github-env-injection (severity: high)

Attacker-controlled input values are written to GitHub special environment files without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

(1) In the 'Setting Environment Variables' step: `inputs.rustflags` is mapped to the env var `NEW_RUSTFLAGS`, then written directly to $GITHUB_ENV via `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV`. A malicious value containing newlines could inject arbitrary environment variables.

(2) In the 'flags' step: `inputs.toolchain` and `inputs.components` are interpolated directly into `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` without sanitization, allowing newline injection into $GITHUB_OUTPUT.

Locations:

- `action.yml:117`
- `action.yml:104`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed three security issues in action.yml:
1. unsafe-shell (line 131): Replaced `curl | sh` with download-to-tempfile pattern using mktemp, chmod +x, execute, then rm -f cleanup.
2. script-injection (lines 127, 104): Moved `github.action_path` into an ACTION_PATH env var for the problem matcher step; replaced the inline `${{contains(inputs.toolchain, 'nightly') && inputs.components && ...}}` expression in the flags step with pure bash conditional logic using already-bound env vars.
3. github-env-injection (lines 117, 104): Added `printf '%s' ... | tr -d '\n\r'` sanitization before writing RUSTFLAGS to $GITHUB_ENV, and before writing targets/components/downgrade to $GITHUB_OUTPUT.

