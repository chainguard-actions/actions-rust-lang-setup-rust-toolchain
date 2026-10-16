<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.3** was hardened automatically. 7 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The `flags` step's `run:` block directly interpolates a GitHub Actions expression inside the shell command string: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`. The expression `${{contains(inputs.toolchain, ...)}}` is expanded by the Actions runner before the shell ever sees it, allowing an attacker-controlled `inputs.toolchain` or `inputs.components` value to inject arbitrary shell syntax.

Locations:

- `action.yml:102`

### script-injection (severity: high)

Sub-rule (b): The `flags` step's `run:` block uses unquoted shell variable expansions of workflow-controllable data. The variables `$t` and `$c` (derived from `inputs.target` and `inputs.components` via the `targets` and `components` env vars) are expanded without double-quotes inside `for` loops: `echo -n ' --target' $t` and `echo -n ' --component' $c`. An attacker-supplied value containing shell metacharacters (`;`, `|`, `&`, etc.) would be interpreted by the shell.

Locations:

- `action.yml:100`
- `action.yml:101`

### github-env-injection (severity: high)

The `Setting Environment Variables` step writes the caller-controlled `inputs.rustflags` value to `$GITHUB_ENV` without sanitization. The value is placed into the `NEW_RUSTFLAGS` env var (`env: NEW_RUSTFLAGS: ${{inputs.rustflags}}`), then written directly: `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV`. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write. A newline embedded in `inputs.rustflags` could inject arbitrary environment variable assignments into subsequent steps.

Locations:

- `action.yml:121`

### script-injection (severity: high)

Sub-rule (a): The `Install Rust Problem Matcher` step's `run:` block directly interpolates `${{ github.action_path }}` inside the shell command string: `echo "::add-matcher::${{ github.action_path }}/rust.json"`. Any `${{ ... }}` expression inside a `run:` block is a script-injection risk because the value is substituted into the shell command string before the shell parses it.

Locations:

- `action.yml:128`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of `curl` directly to `sh`: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote URL is compromised or the connection is intercepted, arbitrary code would be executed immediately in the runner shell. The script should be downloaded to a file, its integrity verified, and then executed separately.

Locations:

- `action.yml:133`

### script-injection (severity: high)

Sub-rule (a): The `rustup toolchain install` step's `run:` block directly interpolates three `${{ steps.flags.outputs.* }}` expressions inside the shell command string: `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`. These step outputs originate from `inputs.target`, `inputs.components`, and `inputs.toolchain`, which are caller-controlled. Injecting shell metacharacters into these inputs would allow command injection.

Locations:

- `action.yml:162`

### script-injection (severity: high)

Sub-rule (a): The `Downgrade registry access protocol when needed` step's `run:` block directly interpolates `${{steps.versions.outputs.rustc-version}}` inside the shell command string used in a bash conditional: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ... ]]`. Although `rustc --version` output is typically trusted, any `${{ ... }}` expression inside a `run:` block is a script-injection risk as the value is substituted before the shell parses it.

Locations:

- `action.yml:183`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all 7 findings in action.yml:
1. (script-injection, line 102) Replaced ${{contains(inputs.toolchain,'nightly') && inputs.components && ...}} with a bash if/else block using the toolchain env var (also added toolchain to the flags step's env block).
2. (script-injection, lines 100-101) Quoted $t and $c as "$t" and "$c" in the for loops that build --target and --component flags.
3. (github-env-injection, line 121) Sanitized $NEW_RUSTFLAGS with 'printf "%s" "$NEW_RUSTFLAGS" | tr -d '\n\r'' before writing to $GITHUB_ENV.
4. (script-injection, line 128) Moved ${{ github.action_path }} into env var ACTION_PATH and referenced it as $ACTION_PATH in the shell command.
5. (unsafe-shell, line 133) Replaced 'curl ... | sh -s -- --default-toolchain none -y' with: download to mktemp file, execute with 'sh "$RUSTUP_INIT" --default-toolchain none -y' (dropping the '--' shell option terminator per instructions), then rm the temp file.
6. (script-injection, line 162) Moved ${{steps.flags.outputs.targets}}, ${{steps.flags.outputs.components}}, and ${{steps.flags.outputs.downgrade}} into env vars flags_targets, flags_components, flags_downgrade and referenced them as shell variables.
7. (script-injection, line 183) Moved ${{steps.versions.outputs.rustc-version}} into env var RUSTC_VERSION and referenced it as $RUSTC_VERSION in the bash conditional.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed both findings in action.yml:

1. github-env-injection: The flags step now sanitizes targets and components outputs before writing to $GITHUB_OUTPUT. Values are built up in intermediate variables (targets_flags, components_flags) and then sanitized with `printf '%s' "$var" | tr -d '\n\r'` before echoing to $GITHUB_OUTPUT.

2. script-injection: All unquoted expansions of user-controlled values have been fixed:
   - flags step: Replaced unquoted `${targets//,/ }` and `${components//,/ }` in for-loop `in` clauses with `IFS=',' read -ra` array splitting and `"${array[@]}"` expansion.
   - rustup toolchain install step: Used `IFS=',' read -ra` for components, targets, and toolchain; used xargs-based tokenization for flags_targets, flags_components, and flags_downgrade step outputs; replaced `${toolchain//*,/ }` with `"${_toolchains[-1]}"` (last array element).

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed all 5 occurrences of `IFS=',' read -ra` that only split on commas. Replaced each with the xargs idiom `_var=(); while IFS= read -r -d '' _t; do _var+=("$_t"); done < <(printf '%s' "${VAR//,/ }" | xargs -r printf '%s\0')` which splits on both commas and whitespace, matching upstream's original shell splitting behavior. Locations fixed: (1) flags step targets split, (2) flags step components split, (3) toolchain install step components split, (4) toolchain install step targets split, (5) toolchain install step toolchains split.

