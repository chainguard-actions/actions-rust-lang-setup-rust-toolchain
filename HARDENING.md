<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.16.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.16.1** was hardened automatically. 7 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): The `flags` step directly interpolates `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` (which includes `inputs.*` values) inside a `run:` shell command string. Any `${{ }}` expression inside a `run:` block is a script-injection risk because the value is substituted into the shell script before the shell parses it. Offending line: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`

Locations:

- `action.yml:107`

### script-injection (severity: high)

Rule (a): The `Install Rust Problem Matcher` step directly interpolates `${{ github.action_path }}` inside a `run:` shell command string: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"`. Even though `github.action_path` is not attacker-controlled, any `${{ }}` expression inside a `run:` block is a script-injection finding per the check rules.

Locations:

- `action.yml:133`

### script-injection (severity: high)

Rule (a): The `rustup toolchain install` step directly interpolates `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` (all `steps.*.outputs.*` context values) inside a `run:` shell command string. Offending line: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`

Locations:

- `action.yml:168`

### script-injection (severity: high)

Rule (a): The `Downgrade registry access protocol when needed` step directly interpolates `${{steps.versions.outputs.rustc-version}}` (a `steps.*.outputs.*` context value) inside a `run:` shell command string used in a bash conditional: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...`. This substitutes the value into the shell script before the shell parses it.

Locations:

- `action.yml:193`

### github-env-injection (severity: high)

The `flags` step writes a value derived from `inputs.toolchain` and `inputs.components` directly to `$GITHUB_OUTPUT` without sanitization: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`. An attacker-controlled input containing newlines could inject additional key=value pairs into GITHUB_OUTPUT. The required sanitization step (`printf '%s' ... | tr -d '\n\r'`) is absent.

Locations:

- `action.yml:107`

### github-env-injection (severity: high)

The `Setting Environment Variables` step writes `$_srt_NEW_RUSTFLAGS` (sourced from `inputs.rustflags` via the `env:` block) to `$GITHUB_ENV` without sanitization: `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV`. A calling workflow can supply a value containing newlines to inject arbitrary environment variables. The required sanitization step (`printf '%s' "$_srt_NEW_RUSTFLAGS" | tr -d '\n\r'`) is absent before the write.

Locations:

- `action.yml:121`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of `curl` directly to `sh` without first saving the script to a file and verifying it: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote URL is compromised or the connection is intercepted, arbitrary code will be executed on the runner.

Locations:

- `action.yml:136`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all 7 findings in hardened/action/action.yml:
1. script-injection (flags step, line 107): Moved `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` to `_srt_downgrade` env var.
2. github-env-injection (flags step, line 107): Added `printf '%s' "$_srt_downgrade" | tr -d '\n\r'` sanitization before writing to GITHUB_OUTPUT.
3. github-env-injection (Setting Environment Variables step, line 121): Added `printf '%s' "$_srt_NEW_RUSTFLAGS" | tr -d '\n\r'` sanitization before writing RUSTFLAGS to GITHUB_ENV.
4. script-injection (Install Rust Problem Matcher step, line 133): Moved `${{ github.action_path }}` to `_srt_action_path` env var.
5. unsafe-shell (Install rustup step, line 136): Download installer to temp file via `mktemp`, execute with `sh "$_srt_rustup_init" --default-toolchain none -y` (dropped `--` since no longer piping to sh -s), then remove temp file.
6. script-injection (rustup toolchain install step, line 168): Moved `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` to `_srt_flags_targets`, `_srt_flags_components`, `_srt_flags_downgrade` env vars.
7. script-injection (Downgrade registry access protocol step, line 193): Moved `${{steps.versions.outputs.rustc-version}}` to `_srt_rustc_version` env var.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed all four findings in action.yml:

1. flags step (lines 116-117) - github-env-injection: Added sanitization of targets and components outputs using `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_OUTPUT.

2. flags step (lines 116-117) - script-injection: Changed `echo -n ' --target' $t` to `printf ' --target %s' "$t"` to properly quote the loop variable.

3. rustup toolchain install step (lines 211-226) - script-injection: Replaced all unquoted variable expansions with xargs-based array tokenization using the `while IFS= read -r -d '' _t; do arr+=("$_t"); done < <(printf '%s' "$VAR" | xargs printf '%s\0')` idiom. All arrays are expanded with `"${arr[@]}"`. Also quoted the `-n` tests for `$_srt_components` and `$_srt_targets`.

4. Downgrade registry step (line 258) - github-env-injection: Added `_srt_rustc_version_safe="$(printf '%s' "$_srt_rustc_version" | tr -d '\n\r')"` and used the sanitized variable in the regex condition.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted shell variable expansions in the `flags` step of action.yml. Replaced `for t in ${_srt_targets//,/ }` and `for c in ${_srt_components//,/ }` (which allowed word splitting and glob expansion on attacker-controlled inputs) with `IFS=',' read -ra _srt_target_arr <<< "$_srt_targets"` / `for t in "${_srt_target_arr[@]}"` and the equivalent for components. This splits on commas safely into arrays and iterates with proper double-quoting, preventing shell metacharacter injection. The step uses `shell: bash` so bash array syntax is valid.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed action.yml lines 116 and 120: replaced `IFS=',' read -ra _srt_target_arr <<< "$_srt_targets"` and `IFS=',' read -ra _srt_component_arr <<< "$_srt_components"` (which split only on commas) with the xargs idiom that substitutes commas with spaces first (`${VAR//,/ }`), then uses `xargs printf '%s\0'` to split on both commas and whitespace, matching upstream's original behavior. Each block is guarded with `if [[ -n "$VAR" ]]` to prevent empty-value issues.

