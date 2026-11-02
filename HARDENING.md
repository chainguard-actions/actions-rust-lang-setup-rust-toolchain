<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v2.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v2.0.1** was hardened automatically. 9 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Step 'flags': (a) the line `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` puts an expression directly inside the run: script. (b) The input-derived env vars are expanded without quotes: `for t in ${_srt_targets//,/ }; do echo -n ' --target' $t; done` and `for c in ${_srt_components//,/ }; ... $c`.

Locations:

- `action.yml:123`
- `action.yml:124`
- `action.yml:125`

### script-injection (severity: high)

Step 'Install Rust Problem Matcher': (a) the line `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` puts a github.* expression directly inside the run: script.

Locations:

- `action.yml:165`

### script-injection (severity: high)

Step 'rustup toolchain install': (a) the line `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update` puts steps.*.outputs expressions directly into the script, and those outputs come from inputs.target and inputs.components. (b) Input-derived env vars are expanded without quotes: `rustup component add ${_srt_components//,/ }`, `rustup target add ${_srt_targets//,/ }`, `rustup toolchain install ${_srt_toolchain//,/ }`, `rustup override set ${_srt_toolchain//*,/ }`, plus `[[ -n $_srt_components ]]` and `[[ -n $_srt_targets ]]`.

Locations:

- `action.yml:212`
- `action.yml:213`
- `action.yml:216`
- `action.yml:223`
- `action.yml:227`

### script-injection (severity: high)

Step 'Downgrade registry access protocol when needed': (a) the line `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ... ]]` puts a steps.*.outputs expression directly inside the run: script.

Locations:

- `action.yml:256`

### script-injection (severity: high)

Step 'Setting Environment Variables': (b) env vars taken from inputs are expanded without quotes inside test expressions: `[[ ( ! -v RUSTFLAGS ) && $_srt_NEW_RUSTFLAGS != "" ]]` and `[[ ( ! -v CARGO_BUILD_WARNINGS ) && $_srt_CARGO_BUILD_WARNINGS != "" ]]`.

Locations:

- `action.yml:146`
- `action.yml:149`

### github-env-injection (severity: high)

Step 'flags': values built from inputs.target, inputs.components and inputs.toolchain (via _srt_targets, _srt_components and an inline expression) are written to $GITHUB_OUTPUT without `printf '%s' ... | tr -d '\n\r'` sanitization.

Locations:

- `action.yml:123`
- `action.yml:124`
- `action.yml:125`

### github-env-injection (severity: high)

Step 'Setting Environment Variables': `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` and `echo "CARGO_BUILD_WARNINGS=$_srt_CARGO_BUILD_WARNINGS" >> $GITHUB_ENV` write values from inputs.rustflags and inputs.build-warnings to GITHUB_ENV without newline sanitization. A newline in either input could inject extra environment variables.

Locations:

- `action.yml:147`
- `action.yml:150`

### github-env-injection (severity: high)

Step 'Install rustup, if needed': `echo "${CARGO_HOME:-$USERPROFILE/.cargo}/bin" | sed ... >> $GITHUB_PATH` and `echo "${CARGO_HOME:-$HOME/.cargo}/bin" >> $GITHUB_PATH` write CARGO_HOME, which comes from the calling workflow's environment, to GITHUB_PATH without `tr -d '\n\r'` sanitization.

Locations:

- `action.yml:175`
- `action.yml:177`

### unsafe-shell (severity: high)

Step 'Install rustup, if needed' pipes remote content straight into a shell: `curl --proto '=https' --tlsv1.2 ... -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. The script should be saved to a file first and then run as a separate step.

Locations:

- `action.yml:171`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

All changes are in action.yml. Nothing was run, so none of this has been tested.

- **flags step:** The inline downgrade check moved into an env var (_srt_downgrade). Targets and components are now split into bash arrays the same way upstream split them: commas replaced by spaces, then split on whitespace using `read -r -d '' -a ... || true`, with an empty-value check first. All three outputs have newlines stripped before they go to GITHUB_OUTPUT.
- **rustup toolchain install step:** This step no longer reads `${{steps.flags.outputs.*}}`. It builds the same arrays from its own env vars and then runs `rustup toolchain install`, `component add`, `target add` and `override set` with quoted array arguments. The flags step still writes its outputs so the action looks the same from outside, but nothing reads them any more.
- **Setting Environment Variables:** The test expressions are now quoted. RUSTFLAGS and CARGO_BUILD_WARNINGS have newlines stripped before they are written to GITHUB_ENV.
- **Problem matcher:** github.action_path now comes in through an env var.
- **Install rustup:** The installer is downloaded to a temp file with `curl -o`, then run with `sh "$f" --default-toolchain none -y` (the `-s --` from the pipe form is dropped). The CARGO_HOME bin path has newlines stripped before it is written to GITHUB_PATH, on both Windows and other runners.
- **Downgrade registry step:** The rustc-version step output now comes in through an env var.

One difference from upstream: the old unquoted expansions also expanded glob characters like `*`, and the new array splitting doesn't. Normal target, component and toolchain names aren't affected.

### Iteration 2

**Fixes applied:** script-injection, unsafe-shell, github-env-injection, invalid-yaml

**Notes:**

I rewrote action.yml in full. A previous pass had garbled the 'rustup toolchain install' and 'versions' steps, and my first targeted edits made that section worse, so patching it piecemeal wasn't reliable.

Changes:
- Problem matcher step: github.action_path now comes in through env (_srt_ACTION_PATH) instead of being written into the run command.
- Install rustup step: the rustup installer is downloaded to a temp file and then run with 'sh "$f" --default-toolchain none -y'. The '-s --' from the old pipe form is dropped. The CARGO_HOME bin path has newlines stripped (printf '%s' | tr -d '\n\r') before it is written to $GITHUB_PATH, on both Windows and other runners.
- Toolchain step: I removed the leftover corrupted code that put step outputs into the command and expanded inputs unquoted. Toolchains, targets and components are now split into bash arrays on commas and whitespace, the same way upstream's unquoted ${x//,/ } split them. The 'rustup override set' argument is the last toolchain in the list, read into an array. I restored the rust-toolchain file branch: '-z toolchain && (-f rust-toolchain || -f rust-toolchain.toml)'.
- versions step: rebuilt with its working-directory and the rustc-version, cargo-version, rustup-version and cachekey outputs.
- Downgrade step: steps.versions.outputs.rustc-version now comes in through env as _srt_RUSTC_VERSION.
- I removed the 'flags' step. Its outputs were only used by the deleted corrupted command, so nothing reads them now.

One line is my reconstruction, not copied from upstream: the rust-toolchain file branch runs 'rustup show active-toolchain || rustup toolchain install'. The corrupted file didn't contain the original command, so I used my best recollection of the later upstream version, which handles rustup 1.28 no longer installing on 'rustup show'. Please check it against upstream v2.0.1.

