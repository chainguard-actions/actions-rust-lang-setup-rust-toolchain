<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.4** was hardened automatically. 8 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): The `flags` step directly interpolates `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` inside the `run:` shell command string on the `echo "downgrade=..."` line. Any `${{ }}` expression inside a `run:` block is subject to YAML template substitution before the shell sees it, enabling script injection. Additionally, rule (b): the unquoted shell expansions `$t` (in `echo -n ' --target' $t`) and `$c` (in `echo -n ' --component' $c`) inside the for-loops expand values derived from `inputs.target` and `inputs.components` without double-quoting, allowing shell metacharacter injection.

Locations:

- `action.yml:107`
- `action.yml:108`
- `action.yml:109`

### script-injection (severity: high)

Rule (b): The `Setting Environment Variables` step uses `$NEW_RUSTFLAGS` unquoted in the condition `$NEW_RUSTFLAGS != ""` (line 129), where `NEW_RUSTFLAGS` is sourced from `inputs.rustflags`. Unquoted expansion of a workflow-controllable env var allows shell metacharacter injection.

Locations:

- `action.yml:129`

### script-injection (severity: high)

Rule (a): The `Install Rust Problem Matcher` step directly interpolates `${{ github.action_path }}` inside the `run:` shell command string: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"`. The `github.action_path` value flows through YAML template substitution before the shell processes it.

Locations:

- `action.yml:145`

### script-injection (severity: high)

Rule (b): The `rustup toolchain install` step uses unquoted shell expansions of `$components` (in `if [[ -n $components ]]` and `rustup component add ${components//,/ }`) and `$targets` (in `if [[ -n $targets ]]` and `rustup target add ${targets//,/ }`), where these variables hold values from `inputs.components` and `inputs.target`. Unquoted expansions allow shell metacharacter injection. Additionally, rule (a): the same step directly interpolates `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` inside the `rustup toolchain install` shell command string.

Locations:

- `action.yml:192`
- `action.yml:193`
- `action.yml:195`
- `action.yml:196`
- `action.yml:203`

### script-injection (severity: high)

Rule (a): The `Downgrade registry access protocol when needed` step directly interpolates `${{steps.versions.outputs.rustc-version}}` inside the `run:` shell command string in the `if` condition: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ... ]]`. This expression is substituted by the YAML template engine before the shell processes it, enabling script injection.

Locations:

- `action.yml:235`

### github-env-injection (severity: high)

The `flags` step writes values derived from `inputs.target` and `inputs.components` to `$GITHUB_OUTPUT` without sanitization. The for-loop expansions `${targets//,/ }` and `${components//,/ }` (where `targets` and `components` come from `inputs.target` and `inputs.components`) are written directly via `echo "targets=..." >> $GITHUB_OUTPUT` and `echo "components=..." >> $GITHUB_OUTPUT`. Additionally, `echo "downgrade=${{...}}" >> $GITHUB_OUTPUT` writes a direct expression interpolation. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before any of these writes.

Locations:

- `action.yml:107`
- `action.yml:108`
- `action.yml:109`

### github-env-injection (severity: high)

The `Setting Environment Variables` step writes `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV` (line 130), where `NEW_RUSTFLAGS` is sourced from `inputs.rustflags` via the `env:` block. Writing an unsanitized user-controlled input value to `$GITHUB_ENV` without the `printf '%s' ... | tr -d '\n\r'` sanitization step allows an attacker to inject arbitrary environment variable definitions (e.g. by embedding newlines in the input value).

Locations:

- `action.yml:130`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of `curl` directly to `sh` without first saving the script to disk for inspection: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Even with TLS enforced, this pattern executes remotely-fetched code immediately and is flagged as unsafe-shell.

Locations:

- `action.yml:151`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all 8 findings in action.yml:
1. flags step: Moved `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` to env block as `downgrade`; quoted `$t`/`$c` via `printf '%s'`; sanitized all GITHUB_OUTPUT writes with `tr -d '\n\r'`.
2. Setting Environment Variables step: Quoted `$NEW_RUSTFLAGS` in condition; sanitized RUSTFLAGS value with `printf '%s' | tr -d '\n\r'` before writing to GITHUB_ENV.
3. Install Rust Problem Matcher step: Moved `${{ github.action_path }}` to env block as `ACTION_PATH`, referenced as `${ACTION_PATH}` in shell.
4. Install rustup step: Downloads script to `/tmp/rustup-init.sh` first, then executes with `sh /tmp/rustup-init.sh --default-toolchain none -y` (dropped the `--` separator which was the shell's option terminator, not the script's argument).
5. rustup toolchain install step: Moved `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` to env block as `flags_targets`, `flags_components`, `flags_downgrade`; quoted `$components` and `$targets` in conditions.
6. Downgrade registry step: Moved `${{steps.versions.outputs.rustc-version}}` to env block as `RUSTC_VERSION`.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection findings in action.yml:

1. flags step (line 95): Replaced unquoted `${targets//,/ }` and `${components//,/ }` in for-loop command substitutions with safe IFS-based array splitting (`IFS=',' read -ra _targets <<< "$targets"`) and properly quoted array expansion (`"${_targets[@]}"`). Added empty-value guards to avoid processing empty inputs.

2. rustup toolchain install step (lines 158/161/163): 
   - Replaced unquoted `${toolchain//,/ }` with IFS-based array splitting + quoted array expansion
   - Replaced unquoted `$flags_targets`, `$flags_components`, `$flags_downgrade` (pre-built flag strings from the flags step) with xargs-based tokenization into bash arrays with proper empty-value guards
   - Replaced unquoted `${components//,/ }` and `${targets//,/ }` in `rustup component add` and `rustup target add` with IFS-based array splitting + quoted array expansion
   - Replaced `${toolchain//*,/ }` (last element) with `"${_toolchains[-1]}"` (bash array last element access)

All bash array constructs are valid since the action explicitly installs a newer bash on macOS and uses `shell: bash` throughout.

