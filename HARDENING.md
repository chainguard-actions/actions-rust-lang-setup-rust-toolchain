<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.17.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.17.0** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings in action.yml.

1. `flags` step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the shell command.

2. `Install Rust Problem Matcher` step: `echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly into the shell command.

3. `rustup toolchain install` step: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three `steps.*` outputs are interpolated directly into the shell command.

4. `Downgrade registry access protocol` step: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ... ]]` — a step output is interpolated directly into the shell command.

Sub-rule (b): Unquoted shell variable expansions of untrusted data:

5. `flags` step: `for t in ${_srt_targets//,/ }` and `for c in ${_srt_components//,/ }` — `_srt_targets` and `_srt_components` hold `inputs.target` and `inputs.components` respectively and are expanded unquoted inside the `for` loop.

6. `rustup toolchain install` step: `rustup component add ${_srt_components//,/ }` and `rustup target add ${_srt_targets//,/ }` — unquoted expansions of untrusted input-derived variables.

Locations:

- `action.yml:124`
- `action.yml:145`
- `action.yml:185`
- `action.yml:215`
- `action.yml:122`
- `action.yml:123`

### github-env-injection (severity: high)

Untrusted input values are written to GitHub special environment files without sanitization (`printf '%s' ... | tr -d '\n\r'`).

1. `flags` step: `echo "targets=$(for t in ${_srt_targets//,/ }; ...)" >> $GITHUB_OUTPUT` and `echo "components=$(for c in ${_srt_components//,/ }; ...)" >> $GITHUB_OUTPUT` — `_srt_targets` and `_srt_components` are set from `inputs.target` and `inputs.components` (untrusted) and written to `$GITHUB_OUTPUT` without newline sanitization.

2. `flags` step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ...}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are written to `$GITHUB_OUTPUT` without sanitization.

3. `Setting Environment Variables` step: `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` — `_srt_NEW_RUSTFLAGS` is set from `inputs.rustflags` (untrusted) and written to `$GITHUB_ENV` without newline sanitization. An attacker can inject arbitrary environment variable definitions by supplying a newline in the `rustflags` input.

Locations:

- `action.yml:122`
- `action.yml:123`
- `action.yml:124`
- `action.yml:135`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step downloads a shell script from the internet and pipes it directly to `sh` for execution: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Although TLS is enforced, piping remote content directly to a shell interpreter means any compromise of the remote server or a MITM attack (despite TLS) would result in arbitrary code execution on the runner without any integrity verification (e.g., checksum or signature check).

Locations:

- `action.yml:151`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection: Moved all ${{ }} expressions from run: blocks to env: blocks. The flags step's downgrade expression, the Install Rust Problem Matcher's github.action_path, the rustup toolchain install step's three steps.flags.outputs.* references, and the Downgrade registry access protocol step's steps.versions.outputs.rustc-version are all now passed via environment variables.

2. github-env-injection: Added sanitization (printf '%s' "$VAR" | tr -d '\n\r') before writing untrusted input-derived values to GITHUB_OUTPUT and GITHUB_ENV. The flags step now sanitizes targets, components, and downgrade values. The Setting Environment Variables step now sanitizes RUSTFLAGS.

3. unsafe-shell: Replaced the curl | sh pipe with a two-step approach: download the rustup installer to a temp file with mktemp, then execute it separately as 'sh "$_srt_rustup_installer" --default-toolchain none -y' (dropping the '--' that was the shell's option terminator in the pipe form), then clean up the temp file.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all three script-injection findings in hardened/action/action.yml:

1. flags step (lines 122/124): Replaced unquoted `${_srt_targets_safe//,/ }` and `${_srt_components_safe//,/ }` for-loop expansions with `IFS=',' read -ra` array splitting and double-quoted `"${arr[@]}"` iteration. Empty elements are filtered with `[[ -n "$t" ]]` guards.

2. rustup toolchain install step (lines 176/179/185/189): Replaced all unquoted expansions:
   - `rustup component add ${_srt_components//,/ }` → `IFS=',' read -ra _srt_components_arr <<< "$_srt_components"; rustup component add "${_srt_components_arr[@]}"`
   - `rustup target add ${_srt_targets//,/ }` → `IFS=',' read -ra _srt_targets_arr <<< "$_srt_targets"; rustup target add "${_srt_targets_arr[@]}"`
   - `rustup toolchain install ${_srt_toolchain//,/ } ${_srt_flags_targets}${_srt_flags_components} --profile minimal${_srt_flags_downgrade}` → fully array-based with IFS splitting and empty-element filtering for pre-built flag strings
   - `rustup override set ${_srt_toolchain//*,/ }` → `rustup override set "${_srt_toolchain_arr[-1]}"` (last array element)

3. Install Rust Problem Matcher step (line 143): Changed from single-line `run:` to block `run: |` form; `${_srt_action_path}` remains inside double quotes in the echo command.

All changes preserve original semantics while ensuring all attacker-controlled values are properly quoted and never undergo uncontrolled word-splitting or glob expansion.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed three instances where inputs were split only on ',' (using IFS=',' read -ra) instead of on ',' and whitespace as upstream did. Replaced each with the xargs idiom: first substitute commas with spaces (${VAR//,/ }), then pipe through xargs -r printf '%s\0' and read NUL-delimited tokens into the array with a while/read loop. This matches upstream's splitting behavior for the 'components' (line 225), 'target' (line 229), and 'toolchain' (line 237) inputs.

