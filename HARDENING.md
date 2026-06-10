<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.3** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): Direct ${{ }} expression interpolation inside run: blocks. (1) The 'flags' step interpolates ${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}} directly in the shell command written to $GITHUB_OUTPUT. (2) The 'Install Rust Problem Matcher' step interpolates ${{ github.action_path }} directly in a run: echo command. (3) The 'rustup toolchain install' step interpolates ${{steps.flags.outputs.targets}}, ${{steps.flags.outputs.components}}, and ${{steps.flags.outputs.downgrade}} directly into a shell command line. (4) The 'Downgrade registry access protocol' step interpolates ${{steps.versions.outputs.rustc-version}} directly inside a bash conditional. All of these bypass shell quoting and allow expression values to be interpreted as shell syntax before the shell ever sees them.

Locations:

- `action.yml:105`
- `action.yml:131`
- `action.yml:183`
- `action.yml:218`

### github-env-injection (severity: high)

Unsanitized writes of untrusted input values to GitHub special environment files. (1) The 'flags' step writes inputs.target and inputs.components (via env vars $targets and $components) to $GITHUB_OUTPUT using bare echo without the required printf '%s' ... | tr -d newline sanitization. A newline embedded in either input can inject arbitrary key=value pairs into GITHUB_OUTPUT. (2) The 'Setting Environment Variables' step writes inputs.rustflags (via env var $NEW_RUSTFLAGS) to $GITHUB_ENV with echo RUSTFLAGS=$NEW_RUSTFLAGS >> $GITHUB_ENV without sanitization. A newline in the rustflags input can inject arbitrary environment variables into the runner.

Locations:

- `action.yml:103`
- `action.yml:104`
- `action.yml:122`

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes remote content directly to a shell interpreter: curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y. If the remote server or network is compromised, arbitrary code executes on the runner without any integrity verification.

Locations:

- `action.yml:140`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection: Moved all ${{ }} expressions out of run: blocks into env: blocks for the 'flags' step (downgrade expression), 'Install Rust Problem Matcher' step (github.action_path), 'rustup toolchain install' step (steps.flags.outputs.*), and 'Downgrade registry access protocol' step (steps.versions.outputs.rustc-version). Shell scripts now reference plain environment variables.

2. github-env-injection: Added printf '%s' ... | tr -d '\n\r' sanitization before writing to $GITHUB_OUTPUT in the 'flags' step (targets, components, downgrade values) and before writing RUSTFLAGS to $GITHUB_ENV in the 'Setting Environment Variables' step.

3. unsafe-shell: Replaced the curl | sh pipe in 'Install rustup, if needed' with a two-step approach: download the script to /tmp/rustup-init.sh, execute it separately with sh, then remove it.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection vulnerabilities in action.yml:

1. **flags step (~line 108):** Replaced unquoted `${targets//,/ }` and `${components//,/ }` for-loop iterations (with unquoted `$t`/`$c` loop variables) with safe `IFS=',' read -ra target_arr <<< "$targets"` / `IFS=',' read -ra component_arr <<< "$components"` array splitting, iterating with `"${target_arr[@]}"`/`"${component_arr[@]}"` and using `printf ' --target %s' "$t"` / `printf ' --component %s' "$c"` with properly quoted arguments.

2. **rustup toolchain install step (~line 155):** Replaced all unquoted variable expansions:
   - `${toolchain//,/ }` → `IFS=',' read -ra toolchain_list <<< "$toolchain"` + `"${toolchain_list[@]}"`
   - `$flag_targets`, `$flag_components`, `$flag_downgrade` → `read -ra ft_args/fc_args/fd_args <<< "$flag_targets/flag_components/flag_downgrade"` + `"${ft_args[@]}"`/`"${fc_args[@]}"`/`"${fd_args[@]}"`
   - `${components//,/ }` and `${targets//,/ }` → `IFS=',' read -ra component_list/target_list` + `"${component_list[@]}"`/`"${target_list[@]}"`
   - `${toolchain//*,/ }` (last element) → `"${toolchain_list[-1]}"`
   - Fixed unquoted `[[ -n $components ]]` and `[[ -n $targets ]]` to use double quotes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed the 'Install rustup, if needed' step in action.yml. Both the Windows and non-Windows branches now sanitize CARGO_HOME (and USERPROFILE/HOME fallbacks) before writing to $GITHUB_PATH. The fix captures the cargo home path into a `safe_cargo_home` variable via `printf '%s' ... | tr -d '\n\r'`, then builds the full bin path into `safe_path` with another `tr -d '\n\r'` pass, and finally writes to $GITHUB_PATH using `printf '%s\n'`. This prevents newline injection attacks where a calling workflow could set CARGO_HOME to a value containing embedded newlines to inject arbitrary additional entries into $GITHUB_PATH.

