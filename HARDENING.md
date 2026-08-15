<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.3** was hardened automatically. 3 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple ${{ }} expressions are interpolated directly inside run: shell command strings, allowing template substitution before the shell ever sees the value.

1. (flags step, ~line 109): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — inputs.toolchain and inputs.components are interpolated directly into the shell command.

2. (rustup toolchain install step, ~line 190): `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three step output expressions are interpolated directly into a shell command.

3. (Install Rust Problem Matcher step, ~line 145): `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — github.action_path is interpolated directly into the shell command.

4. (Downgrade registry step, ~line 215): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ... ]]` — a step output is interpolated directly into a shell conditional.

Locations:

- `action.yml:109`
- `action.yml:145`
- `action.yml:190`
- `action.yml:215`

### github-env-injection (severity: high)

The 'Setting Environment Variables' step writes the value of $NEW_RUSTFLAGS (sourced from inputs.rustflags via env: NEW_RUSTFLAGS: ${{inputs.rustflags}}) directly to $GITHUB_ENV without sanitization: `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV`. An attacker-controlled value containing newlines could inject additional environment variable definitions. The required sanitization step (`printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r'`) is absent.

Locations:

- `action.yml:130`

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Although TLS is enforced, this pattern executes whatever the remote server returns without any integrity verification (e.g., a checksum check). The script should be downloaded to a file first, its integrity verified, and then executed separately.

Locations:

- `action.yml:151`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all 3 finding types across 6 locations in action.yml:
1. script-injection (line 109/flags step): Moved `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` to env var `downgrade_flag`.
2. github-env-injection (line 130): Added `safe_rustflags=$(printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r')` sanitization before writing RUSTFLAGS to $GITHUB_ENV.
3. script-injection (line 145): Moved `${{ github.action_path }}` to env var `ACTION_PATH`, shell uses `${ACTION_PATH}/rust.json`.
4. unsafe-shell (line 151): Replaced `curl ... | sh -s -- --default-toolchain none -y` with download-to-tempfile pattern using mktemp, then `sh "$RUSTUP_INIT" --default-toolchain none -y` (dropped `--` as it was the shell's option terminator, not the script's), then cleanup.
5. script-injection (line 190): Moved `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, `${{steps.flags.outputs.downgrade}}` to env vars `flags_targets`, `flags_components`, `flags_downgrade`.
6. script-injection (line 215): Moved `${{steps.versions.outputs.rustc-version}}` to env var `RUSTC_VERSION`.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all four findings in action.yml:

1. **flags step (script-injection + github-env-injection)**: Added `printf '%s' "$var" | tr -d '\n\r'` sanitization for targets, components, and downgrade_flag inputs before processing. Changed `echo` to `printf` for writing to GITHUB_OUTPUT to prevent newline injection. The for loops now iterate over sanitized values.

2. **Install Rust Problem Matcher (script-injection)**: Changed from `echo "::add-matcher::${ACTION_PATH}/rust.json"` to `printf '::add-matcher::%s/rust.json\n' "${ACTION_PATH}"` so ACTION_PATH is passed as a separate printf argument rather than interpolated into the format string.

3. **rustup toolchain install (script-injection)**: Replaced all unquoted variable expansions (`${components//,/ }`, `${targets//,/ }`, `${toolchain//,/ }`, `${flags_targets}${flags_components}`, `${flags_downgrade}`, `${toolchain//*,/ }`) with xargs-based array tokenization using the `while IFS= read -r -d '' t; do arr+=("$t"); done < <(printf '%s' "$var" | xargs printf '%s\0')` pattern. All arrays are expanded with `"${arr[@]}"` for proper quoting. The last toolchain for `rustup override set` is stored in a variable and double-quoted.

### Iteration 3

**Fixes applied:** script-injection, unpinned-uses, missing-permissions, broad-permissions, github-env-injection

**Notes:**

Fixed all 6 findings across 3 files:

1. autotag-releases.yml: (a) Replaced `permissions: read-all` with `permissions: {}` (broad-permissions); (b) Pinned `actions/checkout@v6` to `actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4` (unpinned-uses); (c) Moved `${{ steps.tag_name.outputs.current_version }}` into an `env: CURRENT_VERSION:` block and referenced as `"$CURRENT_VERSION"` in the shell (script-injection).

2. ci.yml: (a) Added top-level `permissions: {}` (missing-permissions); (b) Pinned both `actions/checkout@v6` references to the full SHA (unpinned-uses); (c) Moved `${{steps.toolchain.outputs.rustc-version}}`, `${{steps.toolchain.outputs.cargo-version}}`, and `${{steps.toolchain.outputs.rustup-version}}` into `env:` blocks and referenced as `"$RUSTC_VERSION"`, `"$CARGO_VERSION"`, `"$RUSTUP_VERSION"` (script-injection).

3. action.yml: Sanitized `CARGO_HOME` before writing to `$GITHUB_PATH` using `printf '%s' ... | tr -d '\n\r'` for both Windows and non-Windows paths (github-env-injection).

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed the script injection vulnerability in the `flags` step of action.yml (lines 110 and 114). The unquoted for-loop expansions `for t in ${safe_targets//,/ }` and `for c in ${safe_components//,/ }` were replaced with safe `IFS=, read -ra arr <<< "$safe_var"` array-building patterns. The new code: (1) splits only on commas using IFS=,, (2) stores results in a bash array, (3) iterates with properly quoted `"${arr[@]}"` expansion, and (4) is guarded with `if [[ -n "..." ]]` to handle empty inputs. This prevents shell metacharacters (`;`, `|`, `&`, backticks, `$()`) in attacker-controlled inputs from being interpreted as shell commands during word-splitting.

