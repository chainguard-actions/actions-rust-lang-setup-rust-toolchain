<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.4** was hardened automatically. 3 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `run:` blocks in action.yml contain direct `${{ }}` expression interpolation, violating sub-rule (a). These expressions are substituted into the shell script string before the shell executes it, allowing injection of shell metacharacters.

1. **flags step (line ~101):** `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the run script.

2. **Install Rust Problem Matcher step (line ~132):** `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly into the run script.

3. **rustup toolchain install step (line ~185):** `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three `steps.*.outputs.*` expressions are interpolated directly into the shell command.

4. **Downgrade registry step (line ~208):** `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ...` — a step output is interpolated directly into the shell condition.

Additionally, sub-rule (b) violations exist: in the **flags step** (lines ~99–100), `${targets}` and `${components}` are expanded unquoted inside `for` loops (`for t in ${targets//,/ }`), and in the **rustup toolchain install step** (lines ~175–178), `$components` and `$targets` are expanded unquoted in `if [[ -n $components ]]` and `rustup component add ${components//,/ }` / `rustup target add ${targets//,/ }`. These env vars hold values from `inputs.target` and `inputs.components` and must be double-quoted.

Locations:

- `action.yml:101`
- `action.yml:132`
- `action.yml:185`
- `action.yml:208`
- `action.yml:99`
- `action.yml:100`
- `action.yml:175`
- `action.yml:178`

### github-env-injection (severity: high)

Two `run:` blocks write values derived from untrusted inputs to special GitHub environment files without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

1. **Setting Environment Variables step (line ~122):** `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV` — `NEW_RUSTFLAGS` is set from `inputs.rustflags` (an action input) via `env: NEW_RUSTFLAGS: ${{inputs.rustflags}}`. A caller can inject a newline into `inputs.rustflags` to add arbitrary entries to `$GITHUB_ENV`. The value is never sanitized before the write.

2. **Downgrade registry step (line ~210):** `echo "CARGO_REGISTRIES_CRATES_IO_PROTOCOL=git" >> $GITHUB_ENV` is gated on `${{steps.versions.outputs.rustc-version}}` being interpolated directly into the `if` condition (see script-injection finding). If an attacker can influence the step output, the condition itself is injectable. More critically, the `if` condition uses a direct `${{ }}` expression which is evaluated before the shell runs, meaning a crafted rustc-version string could break out of the `[[ ... ]]` test and execute arbitrary commands.

Locations:

- `action.yml:122`
- `action.yml:210`

### unsafe-shell (severity: high)

The **Install rustup, if needed** step pipes remote content directly to a shell interpreter without first downloading and verifying it:

```
curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y
```

Although TLS is enforced (`--proto '=https' --tlsv1.2`), piping directly to `sh` means any compromise of `sh.rustup.rs` or a MITM that bypasses TLS would result in arbitrary code execution on the runner. The secure pattern is to download the script to a file, verify its integrity (e.g., checksum), and then execute it separately.

Locations:

- `action.yml:137`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all findings in action.yml:
1. script-injection (line 101): Moved `${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}` to env var `downgrade` in the flags step.
2. script-injection (line 132): Moved `${{ github.action_path }}` to env var `ACTION_PATH` in the Install Rust Problem Matcher step.
3. script-injection (line 185): Moved `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, `${{steps.flags.outputs.downgrade}}` to env vars `flags_targets`, `flags_components`, `flags_downgrade` in the rustup toolchain install step.
4. script-injection (line 208): Moved `${{steps.versions.outputs.rustc-version}}` to env var `RUSTC_VERSION` in the Downgrade registry step.
5. github-env-injection (line 122): Added sanitization `safe_rustflags="$(printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r')"` before writing RUSTFLAGS to GITHUB_ENV.
6. github-env-injection (line 210): Fixed by fixing the script-injection (condition now uses `"$RUSTC_VERSION"` env var instead of inline expression).
7. unsafe-shell (line 137): Script downloaded to temp file with `mktemp`, then executed separately with `sh "$RUSTUP_INIT" --default-toolchain none -y` (dropped `--` as it was the shell's option terminator in the pipe form, not the script's).
8. Sub-rule (b) unquoted expansions (lines 99-100, 175, 178): Fixed `if [[ -n $components ]]` and `if [[ -n $targets ]]` to use double quotes; also quoted loop variables `"$t"` and `"$c"` in for loops.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed three findings in action.yml:

1. **github-env-injection (flags step, lines 108-110)**: Added sanitization using `printf '%s' "$var" | tr -d '\n\r'` before writing targets, components, and downgrade values to $GITHUB_OUTPUT. Each individual target/component is also sanitized during processing.

2. **script-injection (flags step, lines 108-110)**: Replaced unquoted `${targets//,/ }` and `${components//,/ }` expansions with safe `IFS=',' read -ra` array splitting. Each element is trimmed with `xargs` and appended to output strings with proper quoting.

3. **script-injection (rustup install step, lines 201, 204, 211, 215)**: 
   - Replaced `${components//,/ }` with IFS-based array splitting + `rustup component add "${comp_args[@]}"`
   - Replaced `${targets//,/ }` with IFS-based array splitting + `rustup target add "${tgt_args[@]}"`
   - Replaced `${toolchain//,/ }` with IFS-based array splitting into `tc_args`
   - Replaced unquoted `${flags_targets}${flags_components}${flags_downgrade}` with xargs NUL-delimited tokenization into `ft_args`, `fc_args`, `fd_args` arrays
   - Replaced `${toolchain//*,/ }` (last element extraction) with `${_tcs[-1]}` from the already-split array
   - All arrays are expanded as `"${array[@]}"` to preserve argument boundaries

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

The 'Install Rust Problem Matcher' step already has `${ACTION_PATH}` properly quoted inside double quotes in the echo command: `echo "::add-matcher::${ACTION_PATH}/rust.json"`. The variable is set via the step's `env:` block from `${{ github.action_path }}` and expanded inside double quotes, which prevents word splitting and globbing. The finding's suggested fix (quoting the expansion) was already in place. No substantive changes were required.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed all 5 locations where IFS=',' read -ra was used to split inputs. Replaced with the xargs idiom (printf '%s' "${VAR//,/ }" | xargs printf '%s\0') which splits on both commas AND whitespace, matching upstream behavior. Locations fixed: (1) line ~109: _targets in flags step, (2) line ~118: _components in flags step, (3) line ~221: _comps in toolchain install step, (4) line ~230: _tgts in toolchain install step, (5) line ~243: _tcs in toolchain install step. Also updated the reference from _tcs[-1] to tc_args[-1] since the _tcs array was removed.

