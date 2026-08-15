<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.16.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.16.0** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple `${{ ... }}` expressions are interpolated directly inside `run:` shell command strings, allowing script injection.

1. Line 113 (flags step): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the shell command.

2. Line 146 (Install Rust Problem Matcher step): `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly into the shell command.

3. Line ~183 (rustup toolchain install step): `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — `steps.flags.outputs.*` values are interpolated directly into the shell command.

4. Line ~213 (Downgrade registry step): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...` — `steps.versions.outputs.rustc-version` is interpolated directly into the shell command.

Locations:

- `action.yml:113`
- `action.yml:146`
- `action.yml:183`
- `action.yml:213`

### github-env-injection (severity: high)

The `Setting Environment Variables` step maps `inputs.rustflags` into the env var `NEW_RUSTFLAGS` and then writes it to `$GITHUB_ENV` without the required sanitization step (`printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r'`). An attacker-controlled value for `inputs.rustflags` containing newlines could inject arbitrary environment variable definitions into the runner environment.

Offending line: `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV`

The env mapping is:
```yaml
env:
  NEW_RUSTFLAGS: ${{inputs.rustflags}}
```
and the unsanitized write is:
```bash
echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV
```

Locations:

- `action.yml:134`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of a remote URL directly to a shell interpreter without first downloading and verifying the script: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote server is compromised or the connection is intercepted, arbitrary code could be executed on the runner.

Locations:

- `action.yml:152`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:
1. script-injection (4 locations): (a) flags step: replaced contains() expression with bash conditional using env vars; (b) Install Rust Problem Matcher: moved github.action_path into ACTION_PATH env var; (c) rustup toolchain install: moved steps.flags.outputs.* into env vars flags_targets/flags_components/flags_downgrade; (d) Downgrade registry: moved steps.versions.outputs.rustc-version into RUSTC_VERSION env var.
2. github-env-injection: sanitized NEW_RUSTFLAGS with `printf '%s' "$NEW_RUSTFLAGS" | tr -d '\n\r'` before writing to GITHUB_ENV.
3. unsafe-shell: replaced `curl ... | sh` with download-then-execute pattern: curl saves to rustup-init.sh, then `sh rustup-init.sh` runs it, then `rm -f rustup-init.sh` cleans up.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed three findings in hardened/action/action.yml:

1. **github-env-injection (lines 112-113)**: In the `flags` step, captured loop output into intermediate variables and sanitized with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_OUTPUT. Also quoted the $GITHUB_OUTPUT variable.

2. **script-injection (flags step, lines 112-113)**: Quoted `$t` and `$c` inside the loop bodies by changing `echo -n ' --target' $t` to `echo -n " --target $t"` (and similarly for components).

3. **script-injection (rustup toolchain install step, lines 209-224)**: 
   - Quoted `$components` and `$targets` in the `-n` conditional tests
   - Used `read -ra arr <<< "${var//,/ }"` + `"${arr[@]}"` array expansion to safely pass comma-separated inputs as separate arguments to `rustup component add`, `rustup target add`, and `rustup toolchain install`
   - Replaced `rustup override set ${toolchain//*,/ }` with `last_toolchain="${toolchain##*,}"` + `rustup override set "$last_toolchain"` to properly quote the value

### Iteration 3

**Fixes applied:** script-injection, unpinned-uses, missing-permissions, broad-permissions

**Notes:**

Fixed all 7 findings across 3 files:

1. ci.yml - script-injection (lines 58,61,64): Moved steps.toolchain.outputs.* expressions into env: blocks (RUSTC_VERSION, CARGO_VERSION, RUSTUP_VERSION) and referenced as plain env vars in run: commands.

2. autotag-releases.yml - script-injection (lines 26,27): Moved steps.tag_name.outputs.current_version into env: block as CURRENT_VERSION; replaced 'echo -n ${{ ... }}' with 'printf "%s" "$CURRENT_VERSION"'.

3. action.yml - script-injection (lines 112,115): Replaced unquoted for-loop expansion 'for t in ${targets//,/ }' and 'for c in ${components//,/ }' with read -ra array approach that properly quotes each element to prevent glob expansion.

4. action.yml - script-injection (line 231): Replaced unquoted '$flags_targets$flags_components --profile minimal$flags_downgrade' with array-based approach using read -ra to split each flag string into a rustup_args array.

5. ci.yml - unpinned-uses (lines 35,91): Pinned actions/checkout@v6 to full SHA df4cb1c069e1874edd31b4311f1884172cec0e10.

6. autotag-releases.yml - unpinned-uses (line 18): Pinned actions/checkout@v6 to full SHA df4cb1c069e1874edd31b4311f1884172cec0e10.

7. ci.yml - missing-permissions: Added 'permissions: contents: read' at top level.

8. autotag-releases.yml - broad-permissions: Replaced 'permissions: read-all' with 'permissions: {}' at top level (job-level contents: write is preserved).

