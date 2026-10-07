<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.16.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.16.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): Multiple ${{ }} expressions are interpolated directly inside run: shell command strings in action.yml.

1. 'flags' step (line ~101): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — inputs.toolchain and inputs.components are attacker-controlled and flow through YAML template substitution directly into the shell command before quoting.

2. 'Install Rust Problem Matcher' step (line ~127): `echo "::add-matcher::${{ github.action_path }}/rust.json"` — github.action_path is interpolated directly into the run: block.

3. 'rustup toolchain install' step (line ~162): `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — step outputs (derived from inputs) are interpolated directly into the shell command.

4. 'Downgrade registry access protocol' step (line ~185): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ... ]]` — a step output is interpolated directly into a bash conditional expression.

Locations:

- `action.yml:101`
- `action.yml:127`
- `action.yml:162`
- `action.yml:185`

### github-env-injection (severity: high)

The 'Setting Environment Variables' step writes the value of $NEW_RUSTFLAGS (sourced from inputs.rustflags via `env: NEW_RUSTFLAGS: ${{inputs.rustflags}}`) directly to $GITHUB_ENV without sanitization: `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV`. An attacker-controlled value for inputs.rustflags containing newlines could inject arbitrary environment variables into subsequent steps. The required sanitization step (`printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r'`) is absent.

Locations:

- `action.yml:116`

### unsafe-shell (severity: high)

The 'Install rustup, if needed' step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Even though TLS is enforced, piping a remote script directly to sh without first downloading and verifying it is an unsafe pattern — if the remote server or CDN is compromised, arbitrary code executes immediately on the runner.

Locations:

- `action.yml:131`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection (4 locations): Moved all ${{ }} expressions out of run: shell strings into env: blocks. (a) DOWNGRADE_FLAG env var for the contains() expression in the flags step; (b) ACTION_PATH env var for github.action_path in the problem matcher step; (c) FLAGS_TARGETS, FLAGS_COMPONENTS, FLAGS_DOWNGRADE env vars for step outputs in the rustup install step; (d) RUSTC_VERSION env var for the rustc-version step output in the downgrade step.

2. github-env-injection: Added sanitization of NEW_RUSTFLAGS before writing to $GITHUB_ENV: `safe_rustflags=$(printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r')` then writing `RUSTFLAGS=$safe_rustflags`.

3. unsafe-shell: Replaced `curl ... | sh -s -- --default-toolchain none -y` with a safe download-then-execute pattern using mktemp. The script is downloaded to a temp file, executed as `sh "$RUSTUP_INSTALL_SCRIPT" --default-toolchain none -y` (dropping `-s` and `--` which were shell stdin/option-terminator flags, not script arguments), then the temp file is removed.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed three findings in action.yml:

1. github-env-injection (flags step, lines 108-110): Added `printf '%s' "$VAR" | tr -d '\n\r'` sanitization for `targets`, `components`, and `DOWNGRADE_FLAG` before writing to `$GITHUB_OUTPUT`. Also quoted the `$GITHUB_OUTPUT` path.

2. script-injection (flags step, lines 108-109): Replaced unquoted `${targets//,/ }` and `${components//,/ }` expansions in `for` loops with `IFS=',' read -ra arr <<< "$safe_var"` followed by properly quoted `"${arr[@]}"` expansions.

3. script-injection (rustup toolchain install step, lines 183-193): Fixed all unquoted variable expansions:
   - Added quotes to `if [[ -n $components ]]` and `if [[ -n $targets ]]` tests
   - Replaced `rustup component add ${components//,/ }` with array-based approach using `IFS=',' read -ra comp_arr`
   - Replaced `rustup target add ${targets//,/ }` with array-based approach using `IFS=',' read -ra tgt_arr`
   - Replaced the complex `rustup toolchain install ${toolchain//,/ } $FLAGS_TARGETS$FLAGS_COMPONENTS --profile minimal$FLAGS_DOWNGRADE --no-self-update` with a fully array-based approach using empty-value guards for each flag group
   - Replaced `rustup override set ${toolchain//*,/ }` with `"${toolchain_arr[-1]}"` (last element of the already-split array)

