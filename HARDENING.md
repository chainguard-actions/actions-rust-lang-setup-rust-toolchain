<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.17.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **actions-rust-lang--setup-rust-toolchain/v1.17.0** was hardened automatically. 3 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. This executes arbitrary remote code without first downloading and verifying the script.

Locations:

- `action.yml:129`

### script-injection (severity: high)

Rule (a): Multiple ${{ }} expressions are interpolated directly inside run: shell command strings. (1) flags step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — inputs.toolchain and inputs.components interpolated directly. (2) Install Rust Problem Matcher step: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — github.action_path interpolated directly. (3) rustup toolchain install step: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — steps.flags.outputs.* interpolated directly. (4) Downgrade registry step: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ...` — steps.versions.outputs.rustc-version interpolated directly. Rule (b): Unquoted shell variable expansions of untrusted inputs in the rustup toolchain install step: `rustup component add ${_srt_components//,/ }` and `rustup target add ${_srt_targets//,/ }` are unquoted, sourced from inputs.components and inputs.target respectively, allowing shell metacharacter injection.

Locations:

- `action.yml:101`
- `action.yml:124`
- `action.yml:148`
- `action.yml:151`
- `action.yml:155`
- `action.yml:178`

### github-env-injection (severity: high)

The 'Setting Environment Variables' step writes the value of inputs.rustflags (via env var _srt_NEW_RUSTFLAGS) directly to $GITHUB_ENV without sanitization: `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV`. The required sanitization step (printf '%s' "$_srt_NEW_RUSTFLAGS" | tr -d '\n\r') is absent. A calling workflow can supply a newline-containing rustflags input to inject arbitrary environment variables into subsequent steps.

Locations:

- `action.yml:116`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all three findings in action.yml: (1) unsafe-shell: replaced curl|sh pipe with download-to-tempfile pattern using mktemp, chmod +x, execute, rm; (2) script-injection: moved all ${{ }} expressions from run: shells into env: blocks - flags step downgrade now computed in shell from _srt_toolchain env var, github.action_path moved to _srt_action_path, steps.flags.outputs.* moved to _srt_flags_targets/_srt_flags_components/_srt_flags_downgrade, steps.versions.outputs.rustc-version moved to _srt_rustc_version; also fixed unquoted ${_srt_components//,/ } and ${_srt_targets//,/ } using IFS=',' read -ra arrays with proper double-quoting; (3) github-env-injection: sanitized _srt_NEW_RUSTFLAGS with printf '%s' | tr -d '\n\r' before writing to $GITHUB_ENV.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed three locations in action.yml:

1. 'flags' step (lines 112-113): Rewrote the for loops to build target/component strings with proper quoting, then sanitize with `printf '%s' | tr -d '\n\r'` before writing to $GITHUB_OUTPUT. Also quoted $GITHUB_OUTPUT in all redirections in this step.

2. 'Install Rust Problem Matcher' step (line 135): The `${_srt_action_path}` variable is already inside double quotes in the echo command, so it is properly quoted. No change needed (finding was acknowledged as borderline).

3. 'rustup toolchain install' step (line 183): Replaced the unquoted array construction `($_srt_flags_targets $_srt_flags_components --profile minimal$_srt_flags_downgrade --no-self-update)` with safe `read -ra` splitting into intermediate arrays, then proper quoted array expansion: `("${_srt_arr_targets[@]}" "${_srt_arr_components[@]}" --profile "minimal${_srt_flags_downgrade}" --no-self-update)`.

### Iteration 3

**Fixes applied:** unpinned-uses, script-injection, broad-permissions, missing-permissions

**Notes:**

Fixed all four findings across both workflow files:

1. **unpinned-uses**: Pinned all three `actions/checkout@v7.0.0` references to the full SHA `9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0` with `# v7.0.0` comment.

2. **script-injection**: In autotag-releases.yml, moved `${{ steps.tag_name.outputs.current_version }}` to an `env:` block (`CURRENT_VERSION`) and referenced it as `"$CURRENT_VERSION"` in the shell. In ci.yml, moved the three toolchain output expressions (`rustc-version`, `cargo-version`, `rustup-version`) to `env:` blocks and referenced them as `"$RUSTC_VERSION"`, `"$CARGO_VERSION"`, `"$RUSTUP_VERSION"` in the `run:` commands.

3. **broad-permissions**: Replaced `permissions: read-all` with `permissions: {}` at the top level of autotag-releases.yml. The job-level `permissions: contents: write` remains unchanged.

4. **missing-permissions**: Added `permissions: contents: read` at the top level of ci.yml (the workflow only needs to check out code).

### Iteration 4

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities in the `flags` step of action.yml (lines 121 and 125). The unquoted `for t in ${_srt_targets//,/ }` and `for c in ${_srt_components//,/ }` loop constructs were replaced with safe `IFS=',' read -ra <array> <<< "$<var>"` splitting followed by properly quoted `"${array[@]}"` expansion. This prevents word-splitting and glob expansion on attacker-controlled `inputs.target` and `inputs.components` values. The fix uses the same safe pattern already employed elsewhere in the file for similar comma-separated input handling.

