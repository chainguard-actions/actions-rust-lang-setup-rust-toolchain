<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.15.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.15.2** was hardened automatically. 8 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a) and (b), step 'flags': line 110 puts an expression straight into the run script: echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}". Lines 108-109 also expand input-derived env vars without quotes: `for t in ${targets//,/ }; do echo -n ' --target' $t; done` and the matching loop over ${components//,/ }.

Locations:

- `action.yml:110`
- `action.yml:108`
- `action.yml:109`

### script-injection (severity: high)

Rule (a), step 'Install Rust Problem Matcher': the run script contains an expression directly: run: echo "::add-matcher::${{ github.action_path }}/rust.json".

Locations:

- `action.yml:146`

### script-injection (severity: high)

Rule (a) and (b), step 'rustup toolchain install': line 203 puts step outputs into the script: `rustup toolchain install ${toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`. These outputs come from inputs. The step also expands input-derived variables without quotes: `[[ -n $components ]]`, `rustup component add ${components//,/ }` (line 193), `rustup target add ${targets//,/ }` (line 196), `${toolchain//,/ }` (line 203) and `rustup override set ${toolchain//*,/ }` (line 207).

Locations:

- `action.yml:203`
- `action.yml:193`
- `action.yml:196`
- `action.yml:207`

### script-injection (severity: high)

Rule (a), step 'Downgrade registry access protocol when needed': the run script contains an expression directly: if [[ "${{steps.versions.outputs.rustc-version}}" =~ ... ]].

Locations:

- `action.yml:233`

### github-env-injection (severity: high)

Step 'flags' writes values built from inputs.target, inputs.components and inputs.toolchain to $GITHUB_OUTPUT without sanitizing them first with printf '%s' | tr -d '\n\r'. For example: echo "targets=$(for t in ${targets//,/ }; ...)" >> $GITHUB_OUTPUT.

Locations:

- `action.yml:108`

### github-env-injection (severity: high)

Step 'Setting Environment Variables' writes inputs.rustflags, passed in through env NEW_RUSTFLAGS, to $GITHUB_ENV without sanitization: echo "RUSTFLAGS=$NEW_RUSTFLAGS" >> $GITHUB_ENV. A newline in the input could inject extra environment variables.

Locations:

- `action.yml:131`

### github-env-injection (severity: high)

Step 'Install rustup, if needed' copies CARGO_HOME, an environment variable inherited from the calling workflow, into $GITHUB_PATH without sanitization: echo "${CARGO_HOME:-$HOME/.cargo}/bin" >> $GITHUB_PATH. The Windows branch on line 156 does the same.

Locations:

- `action.yml:158`
- `action.yml:156`

### unsafe-shell (severity: high)

Step 'Install rustup, if needed' pipes remote content straight into a shell: curl --proto '=https' --tlsv1.2 ... -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y. The script should be downloaded to a file first and then run as a separate step.

Locations:

- `action.yml:152`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

All changes are in action.yml. 1) 'flags' step: the downgrade expression in the script is gone. inputs.toolchain now comes in through env, and the downgrade check is done in bash, lowercasing first because contains() ignores case. Targets and components are split into arrays with read -d '' after turning commas into spaces, so they split on the same characters as upstream's unquoted expansion. The values written to GITHUB_OUTPUT have newlines stripped. 2) 'rustup toolchain install' step: the step no longer pastes steps.flags.outputs.* into the script. It builds the rustup argument list from env vars as a bash array, keeping upstream's order (toolchains, --target, --component, --profile minimal, optional --allow-downgrade, --no-self-update). component add, target add and override set now take quoted arrays. 3) The matcher step reads github.action_path from env (ACTION_PATH). 4) The downgrade-protocol step reads the rustc version from env (RUSTC_VERSION). 5) RUSTFLAGS has newlines stripped before it goes to GITHUB_ENV. 6) The CARGO_HOME bin path has newlines stripped before it goes to GITHUB_PATH, on both Windows and the other runners. 7) rustup-init is now downloaded to a mktemp file and run with 'sh "$f" --default-toolchain none -y'. The '-s --' from the old pipe form was dropped. Problem along the way: several batched edit_file calls landed in the wrong places and corrupted the file, so I rewrote it in full from the original with every fix included.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

In action.yml, 'Setting Environment Variables' step: put double quotes around $NEW_RUSTFLAGS in the [[ ]] test, giving "$NEW_RUSTFLAGS" != "". The test behaves the same as before.

