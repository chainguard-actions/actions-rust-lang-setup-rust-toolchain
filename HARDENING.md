<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.2** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `${{ }}` expressions are directly interpolated inside `run:` shell command strings in action.yml (sub-rule a), allowing script injection:

1. `flags` step (line ~107): `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` — `inputs.toolchain` and `inputs.components` are interpolated directly into the shell command.

2. `Install Rust Problem Matcher` step (line ~127): `echo "::add-matcher::${{ github.action_path }}/rust.json"` — `github.action_path` is interpolated directly.

3. `rustup toolchain install` step (line ~163): `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` — step outputs are interpolated directly into the shell command.

4. `Downgrade registry` step (line ~185): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) ...` — step output is interpolated directly into the shell command.

Locations:

- `action.yml:107`
- `action.yml:127`
- `action.yml:163`
- `action.yml:185`

### github-env-injection (severity: high)

Unsanitized user-controlled values are written to GitHub special environment files without the required `printf '%s' ... | tr -d '\n\r'` sanitization step:

1. `Setting Environment Variables` step (line ~120): `echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV` — `NEW_RUSTFLAGS` is set from `inputs.rustflags` (caller-controlled) via env var and written directly to `$GITHUB_ENV` without sanitization. A newline in the value could inject arbitrary environment variables.

2. `flags` step (lines ~105-106): `echo "targets=$(...)" >> $GITHUB_OUTPUT` and `echo "components=$(...)" >> $GITHUB_OUTPUT` — `targets` and `components` env vars are sourced from `inputs.target` and `inputs.components` respectively and written to `$GITHUB_OUTPUT` without sanitization.

Locations:

- `action.yml:120`
- `action.yml:105`
- `action.yml:106`

### unsafe-shell (severity: high)

The `Install rustup, if needed` step pipes the output of `curl` directly to `sh` without first saving to a file for inspection: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. If the remote URL is compromised or the response is tampered with in transit, arbitrary code will be executed on the runner.

Locations:

- `action.yml:134`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection: Moved all four ${{ }} expressions from run: blocks into env: blocks - the 'downgrade' expression in the flags step, 'github.action_path' in the problem matcher step, three 'steps.flags.outputs.*' expressions in the toolchain install step, and 'steps.versions.outputs.rustc-version' in the downgrade registry step.

2. github-env-injection: Added printf '%s' ... | tr -d '\n\r' sanitization for: targets and components before writing to $GITHUB_OUTPUT in the flags step, and NEW_RUSTFLAGS before writing to $GITHUB_ENV in the Setting Environment Variables step.

3. unsafe-shell: Replaced the curl | sh pipe pattern with: download to /tmp/rustup-init.sh, execute separately with sh /tmp/rustup-init.sh, then remove the temp file.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings in action.yml:

1. flags step (line 107): Replaced unquoted for-loop word-splitting `for t in ${targets//,/ }` and `for c in ${components//,/ }` with safe IFS-based array splitting: `IFS=',' read -ra arr <<< "$var"` followed by properly quoted `"${arr[@]}"` iteration.

2. rustup toolchain install step (line 155): Replaced all unquoted expansions of attacker-controlled inputs:
   - `${components//,/ }` and `${targets//,/ }` → IFS=',' read -ra arrays with quoted expansion
   - `${toolchain//,/ }` → IFS=',' read -ra toolchain_arr with `"${toolchain_arr[@]}"`
   - `${toolchain//*,/ }` (last element trick) → `"${toolchain_arr[-1]}"` (bash array last element)
   - `${flags_targets}`, `${flags_components}`, `${flags_downgrade}` (unquoted) → read -ra arrays with `"${arr[@]}"` (empty arrays expand to nothing, preserving correct argument counts)

