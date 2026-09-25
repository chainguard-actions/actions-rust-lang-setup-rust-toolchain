<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v2.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v2.0.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings in action.yml. This means YAML template substitution injects the values into the shell command before the shell parses them, enabling script injection.

1. `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly.
2. `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly.
3. `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three step output expressions interpolated directly into a shell command.
4. `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...` — step output interpolated directly into a shell conditional.

Locations:

- `action.yml:110`
- `action.yml:144`
- `action.yml:155`
- `action.yml:174`

### github-env-injection (severity: high)

Multiple `run:` blocks write values derived from untrusted inputs to `$GITHUB_OUTPUT` and `$GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

1. The `flags` step writes `_srt_targets` and `_srt_components` (sourced from `inputs.target` and `inputs.components`) to `$GITHUB_OUTPUT` via unquoted shell variable expansion without sanitization.
2. The `flags` step writes `${{contains(inputs.toolchain, ...)}}` directly to `$GITHUB_OUTPUT` without sanitization.
3. The `Setting Environment Variables` step writes `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` where `_srt_NEW_RUSTFLAGS` comes from `inputs.rustflags` — no sanitization applied.
4. The same step writes `echo "CARGO_BUILD_WARNINGS=$_srt_CARGO_BUILD_WARNINGS" >> $GITHUB_ENV` where `_srt_CARGO_BUILD_WARNINGS` comes from `inputs.build-warnings` — no sanitization applied.

An attacker-controlled newline in any of these inputs can inject arbitrary environment variable assignments.

Locations:

- `action.yml:108`
- `action.yml:109`
- `action.yml:110`
- `action.yml:130`
- `action.yml:133`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of `curl` directly to `sh` without first saving the script to a file and verifying it: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote server or network is compromised, arbitrary code is executed immediately.

Locations:

- `action.yml:150`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection: Moved all four ${{ }} expressions out of run: shell strings into env: blocks:
   - `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` → `_srt_downgrade` env var (flags step)
   - `${{ github.action_path }}` → `_srt_action_path` env var (Install Rust Problem Matcher step)
   - `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, `${{steps.flags.outputs.downgrade}}` → `_srt_flags_targets`, `_srt_flags_components`, `_srt_flags_downgrade` env vars (rustup toolchain install step)
   - `${{steps.versions.outputs.rustc-version}}` → `_srt_rustc_version` env var (Downgrade registry step)

2. github-env-injection: Added `printf '%s' "$VAR" | tr -d '\n\r'` sanitization before writing to $GITHUB_OUTPUT and $GITHUB_ENV for: targets, components, downgrade (flags step), RUSTFLAGS and CARGO_BUILD_WARNINGS (Setting Environment Variables step).

3. unsafe-shell: Replaced `curl ... | sh -s -- --default-toolchain none -y` with: download to a mktemp file, execute with `sh "$_srt_rustup_init" --default-toolchain none -y` (dropping the '--' as required), then clean up the temp file.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all unquoted expansions of user-controlled inputs in the 'rustup toolchain install' step of action.yml:

1. `if [[ -n $_srt_components ]]` → `if [[ -n "$_srt_components" ]]` (added quotes)
2. `rustup component add ${_srt_components//,/ }` → split into array with `IFS=',' read -ra _srt_components_arr <<< "$_srt_components"` then `rustup component add "${_srt_components_arr[@]}"`
3. `if [[ -n $_srt_targets ]]` → `if [[ -n "$_srt_targets" ]]` (added quotes)
4. `rustup target add ${_srt_targets//,/ }` → split into array with `IFS=',' read -ra _srt_targets_arr <<< "$_srt_targets"` then `rustup target add "${_srt_targets_arr[@]}"`
5. `rustup toolchain install ${_srt_toolchain//,/ } ${_srt_flags_targets}${_srt_flags_components} --profile minimal${_srt_flags_downgrade} --no-self-update` → split toolchain with `IFS=',' read -ra`, tokenize _srt_flags_* with xargs into arrays, then expand all with proper quoting
6. `rustup override set ${_srt_toolchain//*,/ }` → `rustup override set "${_srt_toolchain_arr[-1]}"` using the last element of the already-split toolchain array

For comma-separated component/target/toolchain values, IFS-based splitting on ',' is used since these values don't contain spaces. For the flags variables (which are space-separated flag strings like '--target foo --target bar'), xargs-based tokenization into arrays is used to properly handle argument boundaries.

