<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.3** was hardened automatically. 7 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): The `flags` step interpolates `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` directly inside the `run:` shell string. The expression — including attacker-controlled `inputs.*` values — is substituted by the Actions runner before the shell ever sees it, enabling command injection. Offending line: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`

Locations:

- `action.yml:110`

### script-injection (severity: high)

Rule (a): The `Install Rust Problem Matcher` step interpolates `${{ github.action_path }}` directly inside the `run:` shell string. Any `${{ ... }}` expression in a run: block is a script-injection risk as it is substituted before the shell parses the command. Offending line: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"`

Locations:

- `action.yml:143`

### script-injection (severity: high)

Rule (a): The `rustup toolchain install` step interpolates `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` directly inside the `run:` shell string. These step outputs are derived from user-controlled `inputs.*` values and are substituted before the shell parses the command. Offending line: `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`

Locations:

- `action.yml:172`

### script-injection (severity: high)

Rule (a): The `Downgrade registry access protocol when needed` step interpolates `${{steps.versions.outputs.rustc-version}}` directly inside the `run:` shell string inside a `[[ ... =~ ... ]]` regex test. Step outputs flow through YAML template substitution before the shell parses the command. Offending line: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...`

Locations:

- `action.yml:192`

### github-env-injection (severity: high)

The `Setting Environment Variables` step writes `$NEW_RUSTFLAGS` — sourced from `inputs.rustflags` via `env: NEW_RUSTFLAGS: ${{inputs.rustflags}}` — to `$GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A calling workflow can supply a newline-containing value to inject arbitrary environment variables. Offending line: `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV`

Locations:

- `action.yml:131`

### github-env-injection (severity: high)

The `flags` step writes `targets` and `components` — sourced from `inputs.target` and `inputs.components` via the step's `env:` block — to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A calling workflow can supply newline-containing values to inject arbitrary output variables. Offending lines: `echo "targets=$(for t in ${targets//,/ }; ...)" >> $GITHUB_OUTPUT` and `echo "components=$(for c in ${components//,/ }; ...)" >> $GITHUB_OUTPUT`

Locations:

- `action.yml:107`
- `action.yml:108`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of `curl` directly to `sh` for execution: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote content or the TLS connection is compromised, arbitrary code executes on the runner. The script should be downloaded to a file, its integrity verified (e.g. via checksum), and then executed separately.

Locations:

- `action.yml:148`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all 7 findings in action.yml:
1. script-injection (flags step, line 110): Moved `contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''` expression into env block as `downgrade:`, referenced as `$downgrade` in shell.
2. script-injection (Install Rust Problem Matcher, line 143): Moved `${{ github.action_path }}` into env block as `ACTION_PATH`, referenced as `${ACTION_PATH}` in shell.
3. script-injection (rustup toolchain install, line 172): Moved `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, `${{steps.flags.outputs.downgrade}}` into env block as `flags_targets`, `flags_components`, `flags_downgrade`, referenced as shell variables.
4. script-injection (Downgrade registry, line 192): Moved `${{steps.versions.outputs.rustc-version}}` into env block as `RUSTC_VERSION`, referenced as `$RUSTC_VERSION` in shell.
5. github-env-injection (Setting Environment Variables, line 131): Added `safe_rustflags=$(printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r')` before writing RUSTFLAGS to GITHUB_ENV.
6. github-env-injection (flags step, lines 107-108): Added sanitization for targets, components, and downgrade values using `printf '%s' ... | tr -d '\n\r'` before writing to GITHUB_OUTPUT.
7. unsafe-shell (Install rustup, line 148): Replaced `curl ... | sh -s -- --default-toolchain none -y` with download-to-tempfile then execute pattern: `curl ... -o "$RUSTUP_INIT_SCRIPT"` followed by `sh "$RUSTUP_INIT_SCRIPT" --default-toolchain none -y` (dropping the `--` shell option terminator), then `rm -f "$RUSTUP_INIT_SCRIPT"`.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings in action.yml:

1. **flags step (lines 108, 111):** Replaced unquoted `for t in ${targets//,/ }` and `for c in ${components//,/ }` with guarded `IFS=',' read -ra arr <<< "$var"` array splits followed by properly quoted `for x in "${arr[@]}"` iterations. Added `if [[ -n "$targets" ]]` and `if [[ -n "$components" ]]` guards to prevent empty-array issues.

2. **rustup toolchain install step (lines 207, 208, 210, 211, 218, 222):** 
   - Added double quotes to `if [[ -n "$components" ]]` and `if [[ -n "$targets" ]]` conditions
   - Replaced `rustup component add ${components//,/ }` with array-based split + `rustup component add "${component_arr[@]}"`
   - Replaced `rustup target add ${targets//,/ }` with array-based split + `rustup target add "${target_arr[@]}"`
   - Replaced `${toolchain//,/ }` with `IFS=',' read -ra toolchain_arr` + `"${toolchain_arr[@]}"`
   - Replaced `${flags_targets}${flags_components}${flags_downgrade}` with guarded `IFS=' ' read -ra` array splits with empty-value guards
   - Replaced `${toolchain//*,/ }` (last element extraction) with `"${toolchain_arr[-1]}"`

All user-controlled inputs are now properly quoted and split via arrays, preventing shell metacharacter injection.

