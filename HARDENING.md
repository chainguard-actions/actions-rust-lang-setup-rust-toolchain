<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.4** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings (sub-rule a), allowing template substitution before the shell parses the command.

1. `flags` step (line ~103): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the shell command.

2. `Install Rust Problem Matcher` step (line ~133): `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly.

3. `rustup toolchain install` step (line ~172): `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three step output expressions are interpolated directly into the shell command.

4. `Downgrade registry access protocol` step (line ~203): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ...` — a step output expression is interpolated directly into the shell command.

Locations:

- `action.yml:103`
- `action.yml:133`
- `action.yml:172`
- `action.yml:203`

### github-env-injection (severity: high)

Unsanitized user-controlled values are written to GitHub special environment files without the required `printf '%s' ... | tr -d '\n\r'` sanitization step.

1. `flags` step (lines ~101-102): `targets` and `components` env vars (sourced from `inputs.target` and `inputs.components`) are written directly to `$GITHUB_OUTPUT` without sanitization: `echo "targets=$(...)" >> $GITHUB_OUTPUT` and `echo "components=$(...)" >> $GITHUB_OUTPUT`. A newline in the input value could inject additional output variables.

2. `Setting Environment Variables` step (line ~121): `NEW_RUSTFLAGS` (sourced from `inputs.rustflags`) is written to `$GITHUB_ENV` without sanitization: `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV`. A newline in `inputs.rustflags` could inject arbitrary environment variables into the runner.

Locations:

- `action.yml:101`
- `action.yml:121`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes remote content directly to a shell interpreter without first downloading and verifying the script: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote server is compromised or the connection is intercepted, arbitrary code would execute on the runner. The script should be downloaded to a file, its integrity verified (e.g. via checksum), and then executed separately.

Locations:

- `action.yml:137`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three security findings in action.yml:

1. script-injection (4 locations):
   - `flags` step: moved `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` to env block as `downgrade`
   - `Install Rust Problem Matcher` step: moved `${{ github.action_path }}` to env block as `ACTION_PATH`
   - `rustup toolchain install` step: moved `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` to env block as `flags_targets`, `flags_components`, `flags_downgrade`
   - `Downgrade registry access protocol` step: moved `${{steps.versions.outputs.rustc-version}}` to env block as `RUSTC_VERSION`

2. github-env-injection (2 locations):
   - `flags` step: added `printf '%s' | tr -d '\n\r'` sanitization for targets, components, and downgrade before writing to $GITHUB_OUTPUT
   - `Setting Environment Variables` step: added `safe_rustflags=$(printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r')` before writing RUSTFLAGS to $GITHUB_ENV

3. unsafe-shell (1 location):
   - `Install rustup, if needed` step: changed from `curl ... | sh -s -- ...` to downloading script to `/tmp/rustup-init.sh`, executing it separately, then removing the temp file

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all three script-injection findings in action.yml:

1. flags step (lines 104/106): Replaced unquoted `${targets//,/ }` and `${components//,/ }` in for-loop word-splitting contexts with safe `IFS=,; read -ra arr <<< "$var"` array splitting with properly quoted `"${arr[@]}"` iteration.

2. Install Rust Problem Matcher step (line 131): Added sanitization of ACTION_PATH via `printf '%s' "$ACTION_PATH" | tr -d '\n\r'` before use, storing in `safe_action_path`.

3. rustup toolchain install step (lines 155/158/163/167): Replaced all unquoted expansions (`${components//,/ }`, `${targets//,/ }`, `${toolchain//,/ }`, `${flags_targets}`, `${flags_components}`, `${flags_downgrade}`, `${toolchain//*,/ }`) with safe array-based splitting using `IFS=, read -ra` and proper `"${arr[@]}"` expansion. Empty-value guards (`[[ -n "$var" ]] && read -ra arr <<< "$var"`) prevent empty-element injection. The last toolchain element is now extracted as `"${tc_arr[-1]}"` instead of the unquoted `${toolchain//*,/ }` pattern.

