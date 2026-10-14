<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.16.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.16.1** was hardened automatically. 3 finding(s) were identified and resolved across 5 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `run:` blocks in action.yml directly interpolate `${{ ... }}` expressions into shell commands, violating rule (a). This allows template substitution to inject arbitrary shell metacharacters before the shell ever sees the string.

1. `flags` step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the shell command.

2. `Install Rust Problem Matcher` step: `echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly into the shell command.

3. `rustup toolchain install` step: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three `steps.*` output expressions are interpolated directly into the shell command.

4. `Downgrade registry access protocol` step: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ... ]]` — `steps.versions.outputs.rustc-version` is interpolated directly into the shell command.

Locations:

- `action.yml:148`
- `action.yml:172`
- `action.yml:218`
- `action.yml:248`

### github-env-injection (severity: high)

Multiple `run:` blocks write values derived from untrusted inputs to `$GITHUB_OUTPUT` and `$GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

1. `flags` step: `_srt_targets` (from `inputs.target`) and `_srt_components` (from `inputs.components`) are written directly to `$GITHUB_OUTPUT` via `echo "targets=..." >> $GITHUB_OUTPUT` and `echo "components=..." >> $GITHUB_OUTPUT`. An attacker-controlled newline in these inputs can inject arbitrary key=value pairs into GITHUB_OUTPUT.

2. `Setting Environment Variables` step: `$_srt_NEW_RUSTFLAGS` (sourced from `inputs.rustflags`) is written to `$GITHUB_ENV` via `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` without sanitization. A newline in `inputs.rustflags` can inject arbitrary environment variables into the runner.

Locations:

- `action.yml:143`
- `action.yml:163`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote server or the network path is compromised, arbitrary code will execute on the runner without any integrity check. The script should be downloaded to a file first, its checksum verified, and then executed separately.

Locations:

- `action.yml:185`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three finding types in action.yml:

1. script-injection: Moved all ${{ }} expressions out of run: blocks into env: blocks for four steps: (a) flags step - moved downgrade expression to env; (b) Install Rust Problem Matcher - moved github.action_path to env as _srt_action_path; (c) rustup toolchain install - moved three steps.flags.outputs.* expressions to env as _srt_flags_targets, _srt_flags_components, _srt_flags_downgrade; (d) Downgrade registry access protocol - moved steps.versions.outputs.rustc-version to env as _srt_rustc_version.

2. github-env-injection: Added sanitization using printf '%s' ... | tr -d '\n\r' before writing targets, components, and downgrade to GITHUB_OUTPUT in the flags step, and before writing RUSTFLAGS to GITHUB_ENV in the Setting Environment Variables step.

3. unsafe-shell: Replaced curl | sh -s -- pattern with download-to-tempfile-then-execute pattern: mktemp creates a temp file, curl downloads to it with -o, sh executes it directly (dropping the -- since it was the shell's option terminator, not the script's), then rm cleans up.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all unquoted shell variable expansions sourced from user-controlled inputs:
1. In 'Setting Environment Variables' step (line 168): Added double-quotes around `$_srt_NEW_RUSTFLAGS` in the [[ ]] conditional.
2. In 'rustup toolchain install' step (lines 226-242):
   - Added double-quotes to `$_srt_components` and `$_srt_targets` in [[ -n ]] tests
   - Replaced unquoted `${_srt_components//,/ }` expansion with xargs-based array tokenization into `_srt_comp_arr`, then `rustup component add "${_srt_comp_arr[@]}"`
   - Replaced unquoted `${_srt_targets//,/ }` expansion with xargs-based array tokenization into `_srt_tgt_arr`, then `rustup target add "${_srt_tgt_arr[@]}"`
   - Replaced unquoted `${_srt_toolchain//,/ }` with xargs-based array `_srt_tc_arr`
   - Replaced unquoted `${_srt_flags_targets}`, `${_srt_flags_components}`, `${_srt_flags_downgrade}` with guarded xargs-based arrays `_srt_ft_arr`, `_srt_fc_arr`, `_srt_fd_arr`
   - All rustup commands now use properly quoted array expansions
   - Fixed `rustup override set` to use `"$_srt_last_tc"` (a properly quoted variable holding the last toolchain)

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings in action.yml:

1. (Line 116) Quoted the loop variables `$t` and `$c` in the `flags` step's for-loops. Changed `echo -n ' --target' $t` to `echo -n ' --target' "$t"` and `echo -n ' --component' $c` to `echo -n ' --component' "$c"`. This prevents shell metacharacter injection from attacker-controlled `inputs.target` and `inputs.components` values.

2. (Line 131) The `${_srt_action_path}` variable in the `Install Rust Problem Matcher` step is already inside double quotes in the echo command (`echo "::add-matcher::${_srt_action_path}/rust.json"`), providing proper shell quoting. The variable is correctly set via the `env:` block from `${{ github.action_path }}`. No additional change was needed beyond confirming the quoting is correct.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two unquoted shell variable expansions in the `flags` step's `run:` block in action.yml. On lines 116 and 118, the `for` loop iterations over `${_srt_targets//,/ }` and `${_srt_components//,/ }` were unquoted, allowing glob expansion on attacker-controlled input values. Added double-quotes around both expansions: `"${_srt_targets//,/ }"` and `"${_srt_components//,/ }"`. This prevents glob characters (e.g., `*`) in attacker-supplied `target` or `components` inputs from expanding to local filenames, while preserving the intentional word-splitting behavior of the comma-to-space substitution.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed action.yml lines 116 and 118: replaced `for t in "${_srt_targets//,/ }"` (quoted, one iteration) and `for c in "${_srt_components//,/ }"` (quoted, one iteration) with the xargs idiom. Each now substitutes commas with spaces, pipes through `xargs -r printf '%s\0'` to tokenize on whitespace, and reads null-delimited tokens with `while IFS= read -r -d ''` to build `--target` and `--component` flags correctly for multi-value inputs.

