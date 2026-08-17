<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.4** was hardened automatically. 3 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings in action.yml, violating sub-rule (a). This allows template-substituted values to be parsed as shell code before the shell ever sees them.

1. `flags` step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — inputs.toolchain and inputs.components are interpolated directly into the shell command.

2. `Install Rust Problem Matcher` step: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — github.action_path is interpolated directly.

3. `rustup toolchain install` step: `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three step output expressions interpolated directly into the shell command.

4. `Downgrade registry access protocol` step: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ...` — step output expression interpolated directly into the shell command.

Locations:

- `action.yml:100`
- `action.yml:128`
- `action.yml:168`
- `action.yml:198`

### github-env-injection (severity: high)

Multiple `run:` steps write values derived from untrusted inputs to special GitHub environment files without the required sanitization (`printf '%s' ... | tr -d '\n\r'`).

1. `flags` step: `echo "targets=$(for t in ${targets//,/ }; ...)" >> $GITHUB_OUTPUT` and `echo "components=$(for c in ${components//,/ }; ...)" >> $GITHUB_OUTPUT` — the `targets` and `components` env vars are set from `inputs.target` and `inputs.components` respectively and written to GITHUB_OUTPUT without sanitization.

2. `flags` step: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — a value derived from inputs is written directly to GITHUB_OUTPUT without sanitization.

3. `Setting Environment Variables` step: `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV` — `NEW_RUSTFLAGS` is set from `inputs.rustflags` and written to GITHUB_ENV without sanitization, allowing newline injection that could define arbitrary environment variables.

Locations:

- `action.yml:98`
- `action.yml:99`
- `action.yml:100`
- `action.yml:120`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Even though TLS is enforced, piping a remote script directly to `sh` without first downloading and verifying it is an unsafe pattern — if the remote server or CDN is compromised, malicious code executes immediately.

Locations:

- `action.yml:133`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection: Moved all four ${{ }} expressions out of run: shell strings into env: blocks:
   - flags step: ${{contains(inputs.toolchain, 'nightly') && ...}} → env.toolchain_contains_nightly
   - Install Rust Problem Matcher step: ${{ github.action_path }} → env.ACTION_PATH
   - rustup toolchain install step: three ${{steps.flags.outputs.*}} expressions → env.flags_targets, env.flags_components, env.flags_downgrade
   - Downgrade registry step: ${{steps.versions.outputs.rustc-version}} → env.RUSTC_VERSION

2. github-env-injection: Added sanitization (printf '%s' "$VAR" | tr -d '\n\r') before all GITHUB_OUTPUT and GITHUB_ENV writes that use values derived from inputs:
   - flags step: targets, components, and downgrade outputs now sanitized
   - Setting Environment Variables step: RUSTFLAGS write now sanitized

3. unsafe-shell: Replaced `curl ... | sh -s -- --default-toolchain none -y` with: download to a mktemp file, execute the file directly (dropping the shell's '--' option terminator as required), then remove the temp file.

### Iteration 2

**Fixes applied:** unpinned-uses, script-injection, missing-permissions, broad-permissions, github-env-injection

**Notes:**

Fixed all 5 findings across ci.yml and autotag-releases.yml:

1. unpinned-uses: Pinned all 3 occurrences of actions/checkout@v6 to SHA d23441a48e516b6c34aea4fa41551a30e30af803 # v6

2. script-injection: (a) In ci.yml, moved steps.toolchain.outputs.{rustc,cargo,rustup}-version expressions into env: blocks, referencing them as $RUSTC_VERSION/$CARGO_VERSION/$RUSTUP_VERSION in run: scripts. (b) In autotag-releases.yml, moved steps.tag_name.outputs.current_version into an env: block as CURRENT_VERSION.

3. missing-permissions: Added top-level 'permissions: contents: read' to ci.yml.

4. broad-permissions: Replaced top-level 'permissions: read-all' with 'permissions: {}' in autotag-releases.yml; the job-level 'contents: write' is preserved.

5. github-env-injection: In autotag-releases.yml 'Get version from tag' step, sanitized GITHUB_REF value with 'printf "%s" | tr -d "\n\r"' before writing to $GITHUB_OUTPUT to prevent newline injection.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed all 6 unquoted variable expansions in the 'rustup toolchain install' step of action.yml:
1. `$components` / `${components//,/ }` (lines 207-208): Now split into array with `IFS=',' read -ra component_list` and expanded as `"${component_list[@]}"`.
2. `$targets` / `${targets//,/ }` (lines 210-211): Now split into array with `IFS=',' read -ra target_list` and expanded as `"${target_list[@]}"`.
3. `${toolchain//,/ }` (line 218): Now split into `toolchain_list` array and used as base of `install_args` array.
4. `${flags_targets}${flags_components}` (line 218): Now tokenized via `xargs printf '%s\0'` into individual array elements in `install_args`.
5. `${flags_downgrade}` (line 218): Now added as quoted element `"$flags_downgrade"` to `install_args`.
6. `${toolchain//*,/ }` (line 222): Now uses `"${toolchain_list[-1]}"` to get the last toolchain element.
All values are now properly double-quoted, preventing word-splitting and glob expansion on attacker-controlled inputs.

### Iteration 4

**Fixes applied:** script-injection

**Notes:**

Fixed the script injection vulnerability in the `flags` step (lines 97 and 100) of hardened/action/action.yml. Replaced unquoted `for t in ${targets//,/ }` and `for c in ${components//,/ }` with safe array-based iteration using `IFS=','` and `read -ra` to split on commas. Also fixed the unquoted `$t` and `$c` loop variables in the `echo -n` commands to be properly double-quoted. The new pattern: `$(IFS=','; read -ra t_arr <<< "$targets"; for t in "${t_arr[@]}"; do echo -n " --target" "$t"; done)` prevents word splitting, glob expansion, and command injection from attacker-controlled input values.

