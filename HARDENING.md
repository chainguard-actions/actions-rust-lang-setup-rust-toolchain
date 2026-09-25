<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.2** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings in action.yml, allowing template substitution before the shell ever sees the value.

1. `flags` step (line ~97): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the shell command.

2. `Install Rust Problem Matcher` step (line ~123): `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly into the shell command.

3. `rustup toolchain install` step (line ~155): `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three `steps.*.outputs.*` expressions are interpolated directly into the shell command.

4. `Downgrade registry access protocol` step (line ~169): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ... ]]` — `steps.versions.outputs.rustc-version` is interpolated directly into the shell command.

Locations:

- `action.yml:97`
- `action.yml:123`
- `action.yml:155`
- `action.yml:169`

### github-env-injection (severity: high)

The `Setting Environment Variables` step writes the value of `$NEW_RUSTFLAGS` (sourced from `inputs.rustflags` via the `env:` block: `NEW_RUSTFLAGS: ${{inputs.rustflags}}`) to `$GITHUB_ENV` without sanitization: `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV`. A calling workflow can supply a value containing newlines to inject arbitrary environment variables. The required sanitization step (`printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r'`) is absent before the write.

Locations:

- `action.yml:113`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote URL is compromised or the connection is intercepted, arbitrary code executes on the runner. The script should be downloaded to a file first, its integrity verified (e.g. via checksum), and then executed separately.

Locations:

- `action.yml:127`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection (4 locations):
   - flags step: Replaced the `${{contains(...)}}` expression with equivalent bash logic using the `toolchain` env var (added to env block).
   - Install Rust Problem Matcher step: Moved `${{ github.action_path }}` to `ACTION_PATH` env var, referenced as `${ACTION_PATH}` in shell.
   - rustup toolchain install step: Moved `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, `${{steps.flags.outputs.downgrade}}` to `FLAGS_TARGETS`, `FLAGS_COMPONENTS`, `FLAGS_DOWNGRADE` env vars.
   - Downgrade registry access protocol step: Moved `${{steps.versions.outputs.rustc-version}}` to `RUSTC_VERSION` env var.

2. github-env-injection: Added `safe_rustflags=$(printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r')` before writing to `$GITHUB_ENV`.

3. unsafe-shell: Replaced `curl ... | sh -s -- args` with download-to-tempfile then execute pattern. The `--` was dropped (as required) since we're no longer piping to `sh -s`.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all three findings in action.yml:

1. **flags step (lines 108-109) - script-injection + github-env-injection**: Added `printf '%s' | tr -d '\n\r'` sanitization for `targets`, `components`, and `toolchain` inputs before processing. Added `set -f`/`set +f` around for-loops to prevent glob expansion of unquoted `${safe_targets//,/ }` and `${safe_components//,/ }`. The sanitized (newline-stripped) values are written to `$GITHUB_OUTPUT`, preventing injection.

2. **rustup toolchain install step (lines 206-221) - script-injection**: Fixed unquoted `[[ -n $components ]]` and `[[ -n $targets ]]` to use double quotes. Replaced all unquoted list expansions (`${components//,/ }`, `${targets//,/ }`, `${toolchain//,/ }`, `${toolchain//*,/ }`, `${FLAGS_TARGETS}`, `${FLAGS_COMPONENTS}`, `${FLAGS_DOWNGRADE}`) with xargs-based bash array tokenization using the `while IFS= read -r -d '' t; do arr+=("$t"); done < <(printf '%s' "$VAR" | xargs printf '%s\0')` pattern, then expanding with `"${arr[@]}"`.

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed the github-env-injection finding in the 'Install rustup, if needed' step of action.yml. The CARGO_HOME, USERPROFILE, and HOME environment variables (inherited from the calling workflow) were written directly to $GITHUB_PATH without sanitization. Fixed by capturing the resolved path into a `safe_cargo_home` variable using `printf '%s' "${CARGO_HOME:-...}" | tr -d '\n\r'` to strip newlines, then writing the sanitized value to $GITHUB_PATH. This applies to both the Windows branch (with sed path separator conversion) and the Unix branch.

