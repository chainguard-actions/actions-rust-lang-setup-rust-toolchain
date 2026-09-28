<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.16.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.16.1** was hardened automatically. 8 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The `flags` step directly interpolates `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` inside a `run:` shell command string and writes the result to $GITHUB_OUTPUT. Any `${{ ... }}` expression inside a `run:` block is a script-injection risk because YAML template substitution happens before the shell ever sees the value. Offending line: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`

Locations:

- `action.yml:110`

### script-injection (severity: high)

Sub-rule (a): The `rustup toolchain install` step directly interpolates `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` inside a `run:` shell command string. These step outputs flow from user-controlled inputs and are substituted by the YAML template engine before the shell parses the command. Offending line: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`

Locations:

- `action.yml:193`

### script-injection (severity: high)

Sub-rule (a): The `Downgrade registry access protocol when needed` step directly interpolates `${{steps.versions.outputs.rustc-version}}` inside a `run:` shell command string (inside a bash conditional). This step output is derived from `rustc --version` output and is substituted by the YAML template engine before the shell parses the command. Offending line: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...`

Locations:

- `action.yml:218`

### script-injection (severity: high)

Sub-rule (a): The `Install Rust Problem Matcher` step directly interpolates `${{ github.action_path }}` inside a `run:` shell command string. Even though `github.action_path` is considered GitHub-controlled, any `${{ ... }}` expression directly inside a `run:` block is a script-injection finding because YAML template substitution occurs before the shell parses the command. Offending line: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"`

Locations:

- `action.yml:141`

### github-env-injection (severity: high)

The `flags` step writes `_srt_targets` (sourced from `inputs.target`) and `_srt_components` (sourced from `inputs.components`) to `$GITHUB_OUTPUT` via `echo` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). An attacker-controlled newline in these inputs can inject arbitrary key=value pairs into the output file. Offending lines: `echo "targets=$(for t in ${_srt_targets//,/ }; ...)" >> $GITHUB_OUTPUT` and `echo "components=$(for c in ${_srt_components//,/ }; ...)" >> $GITHUB_OUTPUT`

Locations:

- `action.yml:108`
- `action.yml:109`

### github-env-injection (severity: high)

The `Setting Environment Variables` step writes `$_srt_NEW_RUSTFLAGS` (sourced from `inputs.rustflags` via the `env:` block) to `$GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). An attacker-controlled newline in `inputs.rustflags` can inject arbitrary environment variable definitions into the runner environment. Offending line: `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV`

Locations:

- `action.yml:130`

### github-env-injection (severity: high)

The `Downgrade registry access protocol when needed` step uses `${{steps.versions.outputs.rustc-version}}` (a step output, which is an untrusted/workflow-controllable value) in a condition that writes `CARGO_REGISTRIES_CRATES_IO_PROTOCOL=git` to `$GITHUB_ENV`. While the written value itself is a literal, the expression is directly interpolated in the `run:` block without sanitization, and the pattern of writing to `$GITHUB_ENV` based on unsanitized step output is a risk. More critically, the direct `${{ }}` interpolation in the run block (see script-injection finding) means the entire shell command is constructed from an unsanitized value before execution.

Locations:

- `action.yml:218`
- `action.yml:220`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Even with TLS enforced, this pattern executes whatever the remote server returns without any integrity verification (e.g., a checksum check). The script should be downloaded to a file first, its checksum verified, and then executed separately.

Locations:

- `action.yml:148`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all 8 findings in action.yml:

1. script-injection (line 110): Moved `contains(inputs.toolchain, ...)` expression into env var `_srt_downgrade` in the flags step.

2. script-injection (line 193): Moved `steps.flags.outputs.targets/components/downgrade` expressions into env vars `_srt_flag_targets`, `_srt_flag_components`, `_srt_flag_downgrade` in the toolchain install step.

3. script-injection (line 218): Moved `steps.versions.outputs.rustc-version` into env var `_srt_rustc_version` in the downgrade step.

4. script-injection (line 141): Moved `github.action_path` into env var `_srt_action_path` in the problem matcher step.

5. github-env-injection (lines 108-109): Added `printf '%s' ... | tr -d '\n\r'` sanitization for targets and components before writing to $GITHUB_OUTPUT.

6. github-env-injection (line 130): Added `printf '%s' ... | tr -d '\n\r'` sanitization for RUSTFLAGS before writing to $GITHUB_ENV.

7. github-env-injection (lines 218/220): Added `printf '%s' ... | tr -d '\n\r'` sanitization for rustc-version before using in the conditional.

8. unsafe-shell (line 148): Replaced `curl ... | sh -s -- args` with download-to-tempfile + execute pattern, dropping the `--` shell option terminator since we're no longer piping to sh -s.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all 6 unquoted variable expansions in the 'rustup toolchain install' step of action.yml:
1. Quoted the -n conditional tests for _srt_components and _srt_targets
2. Replaced unquoted `${_srt_components//,/ }` with IFS-split array + `"${arr[@]}"` expansion for `rustup component add`
3. Replaced unquoted `${_srt_targets//,/ }` with IFS-split array + `"${arr[@]}"` expansion for `rustup target add`
4. Replaced unquoted `${_srt_toolchain//,/ }` with IFS-split array for the toolchain list
5. Replaced unquoted `${_srt_flag_targets}${_srt_flag_components}` and `${_srt_flag_downgrade}` with xargs-tokenized arrays (these are multi-word flag strings like `--target foo --target bar`)
6. Replaced unquoted `${_srt_toolchain//*,/ }` in `rustup override set` with `"${_srt_tc_arr[-1]}"` (last element of the toolchain array)
Also fixed the `--profile minimal${_srt_flag_downgrade}` concatenation bug by separating them into distinct arguments.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection findings in action.yml:

1. flags step (lines 103, 105): Replaced unquoted `for t in ${_srt_targets//,/ }` and `for c in ${_srt_components//,/ }` loops (which allowed shell metacharacter injection via unquoted variable expansion) with `IFS=',' read -ra` array splitting followed by properly quoted array iteration `"${_srt_t_arr[@]}"` / `"${_srt_c_arr[@]}"`.

2. Install Rust Problem Matcher step (line 120): Converted single-line `run:` to a block scalar (`run: |`) to make the double-quoting of `${_srt_action_path}` unambiguous. The variable remains properly double-quoted inside the shell command.

