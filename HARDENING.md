<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.17.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.17.0** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple `run:` blocks in action.yml directly interpolate `${{ }}` expressions inside shell command strings, enabling script injection. (1) The `flags` step interpolates `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` directly in a run: block that writes to $GITHUB_OUTPUT. (2) The `Install Rust Problem Matcher` step interpolates `${{ github.action_path }}` directly in `run: echo "::add-matcher::${{ github.action_path }}/rust.json"`. (3) The `rustup toolchain install` step interpolates `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` directly into a shell command: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`. (4) The `Downgrade registry access protocol` step interpolates `${{steps.versions.outputs.rustc-version}}` directly inside a bash conditional: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ... ]]`. All of these allow YAML template substitution to inject arbitrary shell metacharacters before the shell ever parses the command.

Locations:

- `action.yml:117`
- `action.yml:153`
- `action.yml:196`
- `action.yml:228`

### github-env-injection (severity: high)

The `Setting Environment Variables` step writes the user-controlled input `inputs.rustflags` to `$GITHUB_ENV` via an env var (`_srt_NEW_RUSTFLAGS`) without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). The offending line is: `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV`. An attacker who controls the `rustflags` input can inject newlines to set arbitrary environment variables for subsequent steps, potentially overriding security-sensitive variables like `PATH` or `LD_PRELOAD`.

Locations:

- `action.yml:135`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Although TLS is enforced and the source (sh.rustup.rs) is well-known, this pattern executes whatever the remote server returns without any integrity verification (e.g., a checksum check), making it vulnerable to a compromised CDN or MITM attack. The script should be downloaded to a file, its checksum verified, and then executed separately.

Locations:

- `action.yml:160`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in hardened/action/action.yml:

1. script-injection (4 locations):
   - flags step (line ~117): Moved `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` to env block as `_srt_downgrade`, referenced as `${_srt_downgrade}` in shell.
   - Install Rust Problem Matcher step (line ~153): Moved `${{ github.action_path }}` to env block as `_srt_action_path`, referenced as `${_srt_action_path}` in shell.
   - rustup toolchain install step (line ~196): Moved `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, and `${{steps.flags.outputs.downgrade}}` to env block as `_srt_flags_targets`, `_srt_flags_components`, `_srt_flags_downgrade`, referenced as shell variables.
   - Downgrade registry access protocol step (line ~228): Moved `${{steps.versions.outputs.rustc-version}}` to env block as `_srt_rustc_version`, referenced as `$_srt_rustc_version` in shell.

2. github-env-injection (line ~135): Added sanitization of `_srt_NEW_RUSTFLAGS` using `printf '%s' "$_srt_NEW_RUSTFLAGS" | tr -d '\n\r'` before writing to $GITHUB_ENV.

3. unsafe-shell (line ~160): Replaced `curl ... | sh -s -- --default-toolchain none -y` with downloading to a temp file via `mktemp`, executing with `sh "$_srt_rustup_installer" --default-toolchain none -y` (dropping the `--` shell option terminator as it's not needed when running a file), then removing the temp file.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed the flags step (lines 120-122) to sanitize inputs with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_OUTPUT, and replaced unquoted for-loop expansions with proper IFS-based array splitting. Fixed the rustup toolchain install step (lines 214-229) to quote all variable expansions in test conditions, use IFS-based arrays for component/target/toolchain lists, use xargs-based tokenization for flag strings from step outputs, handle the downgrade flag via a conditional array, and use proper array last-element access instead of the `${var//*,/ }` pattern.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed three IFS=',' read -ra constructs that only split on commas, replacing them with the xargs idiom that first substitutes commas with spaces (${VAR//,/ }) then tokenizes with xargs -r printf '%s\0' in a while read loop. This correctly handles both comma-separated and whitespace-separated values, matching upstream's splitting behavior. The three locations fixed are: (1) _srt_comp_arr for components input, (2) _srt_tgt_arr for target input, and (3) _srt_toolchain_arr for toolchain input, all in the rust-toolchain file install branch of the action.

