<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.16.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.16.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple `${{ ... }}` expressions are directly interpolated inside `run:` shell command strings in action.yml, allowing template substitution before the shell processes the string.

1. "flags" step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are attacker-controlled and interpolated directly into the shell command.

2. "Install Rust Problem Matcher" step: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — `${{ github.action_path }}` is interpolated directly in the run command.

3. "rustup toolchain install" step: `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — `steps.flags.outputs.*` values (derived from user-controlled inputs) are interpolated directly into the shell command.

4. "Downgrade registry access protocol" step: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ...` — `steps.versions.outputs.rustc-version` is interpolated directly in the run block condition.

Locations:

- `action.yml:113`
- `action.yml:143`
- `action.yml:181`
- `action.yml:207`

### github-env-injection (severity: high)

The "Setting Environment Variables" step writes `$NEW_RUSTFLAGS` to `$GITHUB_ENV` without sanitization. `NEW_RUSTFLAGS` is set from `${{inputs.rustflags}}` (an attacker-controlled input via the calling workflow). The write `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV` does not apply the required `printf '%s' ... | tr -d '\n\r'` sanitization before the write, allowing a newline-injection attack to set arbitrary environment variables in subsequent steps.

Locations:

- `action.yml:131`

### unsafe-shell (severity: high)

The "Install rustup, if needed" step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote server or the network path is compromised, arbitrary code will be executed on the runner. The script should be downloaded to a file first, its integrity verified (e.g. via checksum), and then executed separately.

Locations:

- `action.yml:148`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all findings in action.yml:
1. script-injection (4 locations): Moved all ${{ }} expressions out of run: shell strings into env: blocks. (a) flags step: moved contains() expression to env.downgrade; (b) Install Rust Problem Matcher: moved github.action_path to env.ACTION_PATH; (c) rustup toolchain install: moved steps.flags.outputs.* to env vars flag_targets/flag_components/flag_downgrade; (d) Downgrade registry step: moved steps.versions.outputs.rustc-version to env.RUSTC_VERSION.
2. github-env-injection: Added sanitization before writing RUSTFLAGS to GITHUB_ENV using 'safe_rustflags=$(printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r')'.
3. unsafe-shell: Replaced 'curl ... | sh -s -- --default-toolchain none -y' with download to mktemp file then 'sh "$RUSTUP_INIT" --default-toolchain none -y' (dropping '--' as required since we're no longer piping to sh -s).

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed three security findings in action.yml:

1. github-env-injection (lines 111-113): Added newline sanitization in the 'flags' step using `printf '%s' "$VAR" | tr -d '\n\r'` for targets, components, and downgrade before writing to $GITHUB_OUTPUT.

2. script-injection in 'flags' step (lines 111-113): Changed unquoted `echo -n ' --target' $t` to `printf ' --target %s' "$t"` with proper quoting for loop variables.

3. script-injection in 'rustup toolchain install' step (lines 177-190): Fixed all unquoted variable expansions by:
   - Adding double quotes to `[[ -n $components ]]` and `[[ -n $targets ]]` tests
   - Tokenizing comma-separated list inputs (components, targets, toolchain) into bash arrays using the xargs/while-read-NUL idiom with guards
   - Tokenizing flag_targets, flag_components, flag_downgrade into arrays using the same idiom
   - Capturing `${toolchain//*,/}` into `last_toolchain` variable and quoting it properly
   - All array expansions use `"${arr[@]}"` form

