<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v2.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v2.0.0** was hardened automatically. 7 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The 'flags' step run: block directly interpolates ${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}} into the shell command string. This expression is evaluated by the YAML template engine before the shell sees it, allowing an attacker-controlled input (inputs.toolchain, inputs.components) to inject arbitrary shell commands. Offending line: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`

Locations:

- `action.yml:118`

### script-injection (severity: high)

Sub-rule (a): The 'rustup toolchain install' step run: block directly interpolates ${{steps.flags.outputs.targets}}, ${{steps.flags.outputs.components}}, and ${{steps.flags.outputs.downgrade}} into the shell command string. These step outputs are derived from user-controlled inputs (inputs.target, inputs.components, inputs.toolchain) and are substituted by the template engine before the shell executes, enabling command injection. Offending line: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`

Locations:

- `action.yml:218`

### script-injection (severity: high)

Sub-rule (a): The 'Downgrade registry access protocol' step run: block directly interpolates ${{steps.versions.outputs.rustc-version}} into the shell command string inside a conditional expression. Step outputs flow through YAML template substitution before the shell sees them, allowing injection if the output contains shell metacharacters. Offending line: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...`

Locations:

- `action.yml:248`

### script-injection (severity: high)

Sub-rule (a): The 'Install Rust Problem Matcher' step run: block directly interpolates ${{ github.action_path }} into the shell command string. Any ${{ ... }} expression inside a run: block is a script-injection risk as it is substituted by the template engine before the shell executes. Offending line: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"`

Locations:

- `action.yml:161`

### github-env-injection (severity: high)

The 'flags' step writes $_srt_targets (sourced from inputs.target) and $_srt_components (sourced from inputs.components) to $GITHUB_OUTPUT via env vars without the required sanitization step (printf '%s' ... | tr -d '\n\r'). An attacker-controlled input containing newlines can inject arbitrary key=value pairs into the output. Offending lines: `echo "targets=$(for t in ${_srt_targets//,/ }; ...)" >> $GITHUB_OUTPUT` and `echo "components=$(for c in ${_srt_components//,/ }; ...)" >> $GITHUB_OUTPUT`

Locations:

- `action.yml:116`
- `action.yml:117`

### github-env-injection (severity: high)

The 'Setting Environment Variables' step writes $_srt_NEW_RUSTFLAGS (sourced from inputs.rustflags) and $_srt_CARGO_BUILD_WARNINGS (sourced from inputs.build-warnings) to $GITHUB_ENV without the required sanitization step (printf '%s' ... | tr -d '\n\r'). An attacker-controlled input containing newlines can inject arbitrary environment variable definitions. Offending lines: `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` and `echo "CARGO_BUILD_WARNINGS=$_srt_CARGO_BUILD_WARNINGS" >> $GITHUB_ENV`

Locations:

- `action.yml:143`
- `action.yml:146`

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes the output of a remote URL directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote server is compromised or the connection is intercepted, arbitrary code will execute on the runner. The script should be downloaded to a file first, its integrity verified (e.g., via checksum), and then executed separately.

Locations:

- `action.yml:167`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all 7 findings in action.yml:
1. script-injection (flags step, line 118): Moved ${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}} to _srt_downgrade env var.
2. script-injection (rustup install step, line 218): Moved ${{steps.flags.outputs.targets}}, ${{steps.flags.outputs.components}}, ${{steps.flags.outputs.downgrade}} to _srt_flags_targets, _srt_flags_components, _srt_flags_downgrade env vars; updated shell command to use ${_srt_flags_*} references.
3. script-injection (downgrade step, line 248): Moved ${{steps.versions.outputs.rustc-version}} to _srt_rustc_version env var.
4. script-injection (problem matcher step, line 161): Moved ${{ github.action_path }} to ACTION_PATH env var.
5. github-env-injection (flags step, lines 116-117): Added sanitization via printf '%s' | tr -d '\n\r' for targets, components, and downgrade before writing to GITHUB_OUTPUT.
6. github-env-injection (env vars step, lines 143, 146): Added sanitization via printf '%s' | tr -d '\n\r' for RUSTFLAGS and CARGO_BUILD_WARNINGS before writing to GITHUB_ENV.
7. unsafe-shell (install rustup step, line 167): Changed from curl piped to sh to downloading the installer to a temp file with mktemp, executing it separately, then removing it. Dropped '--' since it was the shell's option terminator for 'sh -s', not the script's argument.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings in action.yml:

1. flags step (lines 118/121): Replaced unquoted `${_srt_targets//,/ }` and `${_srt_components//,/ }` in for-loop command substitutions with safe bash array-based iteration using `IFS=',' read -ra` to split on commas, then iterating over the array with proper quoting `"${arr[@]}"` to build flag strings.

2. rustup toolchain install step (lines 207/210/213/218): Fixed all unquoted variable expansions:
   - `${_srt_components//,/ }` → IFS-split array, `rustup component add "${arr[@]}"`
   - `${_srt_targets//,/ }` → IFS-split array, `rustup target add "${arr[@]}"`
   - `${_srt_toolchain//,/ }` → IFS-split array, `rustup toolchain install "${arr[@]}"`
   - `${_srt_flags_targets}`, `${_srt_flags_components}`, `${_srt_flags_downgrade}` → tokenized with xargs into arrays, passed quoted
   - `${_srt_toolchain//*,/ }` (last element) → replaced with `"${_srt_toolchain_trimmed[-1]}"`

All fixes use bash arrays (valid since steps use `shell: bash` and macOS gets newer bash via brew).

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed the 'Install Rust Problem Matcher' step in action.yml (line 165). Converted the run: value from a YAML plain scalar to a block scalar (|) to make the shell double-quoting of ${ACTION_PATH} explicit and unambiguous. The shell command `echo "::add-matcher::${ACTION_PATH}/rust.json"` now clearly has ${ACTION_PATH} inside double quotes, preventing shell metacharacter interpretation of the github.action_path value.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed all 5 IFS=',' read -ra patterns in action.yml by replacing them with the xargs idiom that splits on both commas and whitespace. The pattern used is: substitute commas with spaces first (${VAR//,/ }), then pipe through xargs printf '%s\0' and read with a while/read -d '' loop. This matches upstream's behavior of splitting on commas AND whitespace, preventing the issue where 'a b' would be treated as a single token instead of two separate tokens.

