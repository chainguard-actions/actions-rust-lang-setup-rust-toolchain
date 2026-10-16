<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v2.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v2.0.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings, bypassing shell quoting and enabling script injection.

1. `flags` step (line ~118): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the shell string.

2. `Install Rust Problem Matcher` step (line ~138): `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly into the shell string.

3. `rustup toolchain install` step (line ~196): `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three `steps.*` context expressions are interpolated directly into the shell command.

4. `Downgrade registry access protocol` step (line ~218): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ... ]]` — `steps.versions.outputs.rustc-version` is interpolated directly into the shell string.

Locations:

- `action.yml:118`
- `action.yml:138`
- `action.yml:196`
- `action.yml:218`

### github-env-injection (severity: high)

The `Setting Environment Variables` step writes values derived from `inputs.rustflags` and `inputs.build-warnings` to `$GITHUB_ENV` without the required sanitization (`printf '%s' ... | tr -d '\n\r'`). The values are routed through env vars `_srt_NEW_RUSTFLAGS` and `_srt_CARGO_BUILD_WARNINGS` respectively, but routing through an env var does NOT sanitize the value. An attacker-controlled input containing newlines could inject arbitrary environment variable assignments.

Offending lines:
- `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` (where `_srt_NEW_RUSTFLAGS=${{inputs.rustflags}}`)
- `echo "CARGO_BUILD_WARNINGS=$_srt_CARGO_BUILD_WARNINGS" >> $GITHUB_ENV` (where `_srt_CARGO_BUILD_WARNINGS=${{inputs.build-warnings}}`)

Locations:

- `action.yml:148`
- `action.yml:153`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step downloads and pipes a remote shell script directly to `sh` without first saving it to disk for inspection: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote server is compromised or the connection is intercepted, arbitrary code will execute on the runner.

Locations:

- `action.yml:163`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection (4 locations):
   - flags step: moved `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` to `_srt_downgrade` env var
   - Install Rust Problem Matcher step: moved `${{ github.action_path }}` to `_srt_action_path` env var
   - rustup toolchain install step: moved three `${{steps.flags.outputs.*}}` expressions to `_srt_flags_targets`, `_srt_flags_components`, `_srt_flags_downgrade` env vars
   - Downgrade registry access protocol step: moved `${{steps.versions.outputs.rustc-version}}` to `_srt_rustc_version` env var

2. github-env-injection (2 locations): Added sanitization for RUSTFLAGS and CARGO_BUILD_WARNINGS using `printf '%s' "$VAR" | tr -d '\n\r'` before writing to $GITHUB_ENV.

3. unsafe-shell (1 location): Replaced `curl ... | sh -s -- --default-toolchain none -y` with: download to mktemp file, execute with `sh "$_srt_rustup_init" --default-toolchain none -y` (dropping `--` which was the shell's option terminator), then remove the temp file.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed three findings in action.yml:

1. **script-injection + github-env-injection in 'flags' step**: Replaced unquoted `for t in ${_srt_targets//,/ }` and `for c in ${_srt_components//,/ }` loops with safe xargs-based NUL-delimited tokenization. Each token is read into a loop variable and flag strings are built with properly double-quoted expansions. All three values (targets, components, downgrade) are now sanitized with `printf '%s' ... | tr -d '\n\r'` before being written to `$GITHUB_OUTPUT`.

2. **script-injection in 'rustup toolchain install' step**: Replaced all unquoted variable expansions with xargs-based tokenization into bash arrays:
   - `${_srt_components//,/ }` → `_srt_comp_args` array
   - `${_srt_targets//,/ }` → `_srt_tgt_args` array
   - `${_srt_toolchain//,/ }` → `_srt_toolchain_args` array
   - `${_srt_flags_targets}` and `${_srt_flags_components}` → `_srt_extra_args` array
   - `${_srt_flags_downgrade}` → `_srt_downgrade_args` array
   - `${_srt_toolchain//*,/ }` → `"${_srt_toolchain//*,/}"` (properly quoted)
   All arrays expanded with `"${array[@]}"` to keep each element as a separate properly-quoted argument.

