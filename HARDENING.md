<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.3** was hardened automatically. 3 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings (sub-rule a), allowing template substitution before the shell processes the value:

1. Line 97 (`flags` step): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the shell command.

2. Line 118 (`Install Rust Problem Matcher` step): `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly into the shell command.

3. Line 155 (`rustup toolchain install` step): `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — three `steps.flags.outputs.*` values are interpolated directly into the shell command.

4. Line 172 (`Downgrade registry access protocol` step): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ ...` — `steps.versions.outputs.rustc-version` is interpolated directly into the shell command.

All of these should be moved to `env:` variables and referenced as `"$VAR"` in the shell script.

Locations:

- `action.yml:97`
- `action.yml:118`
- `action.yml:155`
- `action.yml:172`

### github-env-injection (severity: high)

Two steps write values derived from untrusted inputs to GitHub special environment files without the required sanitization (`printf '%s' ... | tr -d '\n\r'`):

1. Line 97 (`flags` step): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — a value derived from `inputs.toolchain` and `inputs.components` is written directly to `$GITHUB_OUTPUT` without sanitization.

2. Line 107 (`Setting Environment Variables` step): `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV` — `NEW_RUSTFLAGS` is set from `inputs.rustflags` via the `env:` block, then written to `$GITHUB_ENV` without sanitization. Routing through an `env:` variable does not sanitize the value; a newline in `inputs.rustflags` could inject arbitrary environment variables.

Locations:

- `action.yml:97`
- `action.yml:107`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of `curl` directly to `sh` without first saving it to a file for inspection:

`curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`

If the remote URL is compromised or the response is tampered with in transit, arbitrary code will be executed on the runner. The script should be downloaded to a temporary file, its integrity verified (e.g. via a checksum), and then executed separately.

Locations:

- `action.yml:124`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection: Moved all four ${{ }} expressions out of run: shell strings into env: blocks. (a) flags step: downgrade expression moved to env:downgrade; (b) Install Rust Problem Matcher: github.action_path moved to env:ACTION_PATH; (c) rustup toolchain install: three steps.flags.outputs.* moved to env:flag_targets/flag_components/flag_downgrade; (d) Downgrade registry step: steps.versions.outputs.rustc-version moved to env:RUSTC_VERSION.

2. github-env-injection: Added sanitization via printf+tr before writing to $GITHUB_OUTPUT and $GITHUB_ENV. The downgrade value in the flags step is sanitized with safe_downgrade before writing to GITHUB_OUTPUT. The RUSTFLAGS value is sanitized with safe_rustflags before writing to GITHUB_ENV.

3. unsafe-shell: Replaced 'curl ... | sh -s -- --default-toolchain none -y' with downloading to a mktemp file first, then executing 'sh "$RUSTUP_INIT_SCRIPT" --default-toolchain none -y' (dropping the '--' as required since we're no longer using sh -s), then removing the temp file.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed two findings in action.yml:

1. github-env-injection (lines 108-109): Captured for-loop output into raw_targets/raw_components variables, then sanitized with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_OUTPUT, matching the existing pattern used for the `downgrade` variable.

2. script-injection (lines 108-109): Quoted `"$t"` and `"$c"` in the echo -n calls within the for loops in the flags step.

3. script-injection (lines 203-218): Fixed unquoted variable expansions in the rustup toolchain install step by:
   - Adding quotes to `[[ -n "$components" ]]` and `[[ -n "$targets" ]]` conditions
   - Using `IFS=',' read -ra arr <<< "$var"` + `"${arr[@]}"` to safely split comma-separated inputs into arrays with each token properly quoted
   - Using `"${tc_arr[-1]}"` instead of `${toolchain//*,/ }` to get the last toolchain element safely

The $flag_targets, $flag_components, and $flag_downgrade variables remain unquoted as they are pre-sanitized outputs from the flags step that need word-splitting to pass as separate arguments to rustup.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed three script-injection findings in hardened/action/action.yml:
1. Lines 97/100 (flags step): Replaced unquoted `${targets//,/ }` and `${components//,/ }` in for loops with safe `IFS=',' read -ra arr <<< "$var"` array splitting and properly quoted `"${arr[@]}"` iteration.
2. Line 130 (Setting Environment Variables step): Added double quotes around `$NEW_RUSTFLAGS` in the `[[ ]]` conditional: `"$NEW_RUSTFLAGS" != ""`.
3. Line 222 (rustup toolchain install step): Replaced unquoted `$flag_targets$flag_components` and `$flag_downgrade` with xargs-based tokenization into bash arrays (`extra_flags` and `downgrade_flags`), expanded safely as `"${extra_flags[@]}"` and `"${downgrade_flags[@]}"`.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Fixed all 5 instances where IFS=',' read -ra arr <<< "$var" was used to split inputs on commas only, but upstream split on commas AND whitespace. Replaced each with the xargs idiom: initialize empty array, then 'while IFS= read -r -d '' t; do arr+=("$t"); done < <(printf '%s' "${var//,/ }" | xargs printf '%s\0')'. This substitutes commas with spaces first, then uses xargs to tokenize on whitespace, correctly handling comma-separated, space-separated, and mixed inputs. Fixed locations: (1) tgt_arr in flags step, (2) cmp_arr in flags step, (3) comp_arr in toolchain install step, (4) tgt_arr in toolchain install step, (5) tc_arr in toolchain install step.

