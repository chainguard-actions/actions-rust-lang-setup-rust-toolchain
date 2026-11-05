<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v2.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v2.0.1** was hardened automatically. 10 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

(a) An expression is inserted straight into the run: script in the 'flags' step: echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT. (b) Variables built from inputs are expanded without quotes: for t in ${_srt_targets//,/ }; do echo -n ' --target' $t; done (the same applies to _srt_components / $c).

Locations:

- `action.yml:123`
- `action.yml:124`
- `action.yml:125`

### script-injection (severity: high)

(b) Variables built from inputs are expanded without quotes inside [[ ]] tests in the 'Setting Environment Variables' step: if [[ ( ! -v RUSTFLAGS ) && $_srt_NEW_RUSTFLAGS != "" ]] and $_srt_CARGO_BUILD_WARNINGS != "".

Locations:

- `action.yml:146`
- `action.yml:149`

### script-injection (severity: high)

(a) An expression is inserted straight into the run: script in the 'Install Rust Problem Matcher' step: run: echo "::add-matcher::${{ github.action_path }}/rust.json".

Locations:

- `action.yml:165`

### script-injection (severity: high)

The 'rustup toolchain install' step breaks both rules. (a) Step outputs are inserted straight into the script: rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update. (b) Variables built from inputs are expanded without quotes: rustup component add ${_srt_components//,/ }, rustup target add ${_srt_targets//,/ }, [[ -n $_srt_components ]], and rustup override set ${_srt_toolchain//*,/ }.

Locations:

- `action.yml:212`
- `action.yml:213`
- `action.yml:216`
- `action.yml:223`
- `action.yml:227`

### script-injection (severity: high)

(a) A step output is inserted straight into the run: script in the 'Downgrade registry access protocol when needed' step: if [[ "${{steps.versions.outputs.rustc-version}}" =~ ...

Locations:

- `action.yml:256`

### github-env-injection (severity: high)

The 'flags' step writes values built from inputs.target and inputs.components (through _srt_targets / _srt_components) and from an expression over inputs.toolchain/inputs.components to $GITHUB_OUTPUT. There is no printf '%s' | tr -d '\n\r' step before the writes.

Locations:

- `action.yml:123`
- `action.yml:124`
- `action.yml:125`

### github-env-injection (severity: high)

The 'Setting Environment Variables' step writes inputs.rustflags and inputs.build-warnings to $GITHUB_ENV (through _srt_NEW_RUSTFLAGS / _srt_CARGO_BUILD_WARNINGS) without stripping newlines: echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV and echo "CARGO_BUILD_WARNINGS=$_srt_CARGO_BUILD_WARNINGS" >> $GITHUB_ENV. A newline in either input lets the caller set any environment variable.

Locations:

- `action.yml:147`
- `action.yml:150`

### github-env-injection (severity: high)

The 'Install rustup, if needed' step writes the inherited, workflow-controlled CARGO_HOME (and USERPROFILE/HOME) env vars to $GITHUB_PATH without stripping newlines: echo "${CARGO_HOME:-$HOME/.cargo}/bin" >> $GITHUB_PATH, plus the same write on the Windows branch.

Locations:

- `action.yml:175`
- `action.yml:177`

### unsafe-shell (severity: high)

Remote content is piped straight into a shell: curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y. The script should be downloaded to a file first and then run as a separate step.

Locations:

- `action.yml:171`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 3178df6..a93373b 100644
--- a/action.yml
+++ b/action.yml
@@ -106,6 +106,23 @@ outputs:
 runs:
   using: composite
   steps:
     # The later code uses "newer" bash features, which are not available in the ancient bash 3 (from 2014) of macOS
     - name: Unbork mac
       if: runner.os == 'macOS'
@@ -113,17 +130,6 @@ runs:
         brew install bash
       shell: bash
 
-    - id: flags
-      env:
-        _srt_targets: ${{inputs.target}}
-        _srt_components: ${{inputs.components}}
-      shell: bash
-      run: |
-        : construct rustup command line
-        echo "targets=$(for t in ${_srt_targets//,/ }; do echo -n ' --target' $t; done)" >> $GITHUB_OUTPUT
-        echo "components=$(for c in ${_srt_components//,/ }; do echo -n ' --component' $c; done)" >> $GITHUB_OUTPUT
-        echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT
-
     # The environment variables always need to be set before the caching action
     - name: Setting Environment Variables
       env:
@@ -143,11 +149,13 @@ runs:
         if [[ ! -v RUST_BACKTRACE ]]; then
           echo "RUST_BACKTRACE=short" >> $GITHUB_ENV
         fi
-        if [[ ( ! -v RUSTFLAGS ) && $_srt_NEW_RUSTFLAGS != "" ]]; then
-          echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV
+        if [[ ( ! -v RUSTFLAGS ) && "$_srt_NEW_RUSTFLAGS" != "" ]]; then
+          _srt_safe=$(printf '%s' "$_srt_NEW_RUSTFLAGS" | tr -d '\n\r')
+          printf 'RUSTFLAGS=%s\n' "$_srt_safe" >> "$GITHUB_ENV"
         fi
-        if [[ ( ! -v CARGO_BUILD_WARNINGS ) && $_srt_CARGO_BUILD_WARNINGS != "" ]]; then
-          echo "CARGO_BUILD_WARNINGS=$_srt_CARGO_BUILD_WARNINGS" >> $GITHUB_ENV
+        if [[ ( ! -v CARGO_BUILD_WARNINGS ) && "$_srt_CARGO_BUILD_WARNINGS" != "" ]]; then
+          _srt_safe=$(printf '%s' "$_srt_CARGO_BUILD_WARNINGS" | tr -d '\n\r')
+          printf 'CARGO_BUILD_WARNINGS=%s\n' "$_srt_safe" >> "$GITHUB_ENV"
         fi
         # Enable faster sparse index on nightly
         # The value is ignored on stable and causes no problems
@@ -162,19 +170,27 @@ runs:
     - name: Install Rust Problem Matcher
       if: inputs.matcher == 'true'
       shell: bash
-      run: echo "::add-matcher::${{ github.action_path }}/rust.json"
+      env:
+        _srt_ACTION_PATH: ${{ github.action_path }}
+      run: |
+        echo "::add-matcher::${_srt_ACTION_PATH}/rust.json"
 
     - name: Install rustup, if needed
       shell: bash
       run: |
         if ! command -v rustup &> /dev/null ; then
-          curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y
+          _srt_installer="$(mktemp)"
+          curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" -o "$_srt_installer"
+          sh "$_srt_installer" --default-toolchain none -y
+          rm -f "$_srt_installer"
 
           # Resolve the correct CARGO_HOME path depending on OS
           if [[ "$RUNNER_OS" == "Windows" ]]; then
-            echo "${CARGO_HOME:-$USERPROFILE/.cargo}/bin" | sed 's|/|\\|g' >> $GITHUB_PATH
+            _srt_cargo_bin=$(printf '%s' "${CARGO_HOME:-$USERPROFILE/.cargo}/bin" | tr -d '\n\r')
+            printf '%s\n' "$_srt_cargo_bin" | sed 's|/|\\|g' >> "$GITHUB_PATH"
           else
-            echo "${CARGO_HOME:-$HOME/.cargo}/bin" >> $GITHUB_PATH
+            _srt_cargo_bin=$(printf '%s' "${CARGO_HOME:-$HOME/.cargo}/bin" | tr -d '\n\r')
+            printf '%s\n' "$_srt_cargo_bin" >> "$GITHUB_PATH"
           fi
         fi
       env:
@@ -187,8 +203,17 @@ runs:
         _srt_components: ${{inputs.components}}
         _srt_override: ${{inputs.override}}
         _srt_rust_src_dir: ${{inputs.rust-src-dir}}
+        _srt_downgrade: ${{ (contains(inputs.toolchain, 'nightly') && inputs.components) && 'true' || '' }}
       shell: bash
       run: |
+        _srt_t_list=()
+        if [[ -n "$_srt_targets" ]]; then
+          read -r -d '' -a _srt_
... (truncated)
```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell, prior-fix

**Notes:**

Removed the 'flags' step, whose outputs were used only by the install step. The install step now builds the rustup command itself as bash arrays: targets, components and toolchains are split the same way as upstream's unquoted ${VAR//,/ } (commas and whitespace), using read -r -d '' -a with || true. The --allow-downgrade decision comes from an env expression (_srt_downgrade). The override step takes the last toolchain with ${_srt_toolchain##*,}. Variables inside [[ ]] tests are now quoted. RUSTFLAGS and CARGO_BUILD_WARNINGS have newlines stripped and are written with printf. github.action_path and steps.versions.outputs.rustc-version are now passed in through env. The rustup installer is downloaded to a mktemp file and run as 'sh file --default-toolchain none -y' (the pipe's -s and -- are dropped). The CARGO_HOME bin paths have newlines stripped before they go to GITHUB_PATH. This re-applies the previous harden's patch.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

All changes are in action.yml. Nothing was run or tested; I only re-read the repaired section to check it.

1. The 'flags' step:
- **Expression in the script:** the `downgrade` value was built by a `${{ }}` expression inside the run script. That expression is now in the step's env block as `_srt_downgrade`, and the script only checks whether it equals 'true'.
- **Unquoted inputs:** targets and components are now split into bash arrays on commas and whitespace, the same way upstream's `${VAR//,/ }` split them.
- **Writing to $GITHUB_OUTPUT:** every value has newlines stripped (`tr -d '\n\r'`) and is written with printf.
- **Output format:** the step still writes `targets`, `components` and `downgrade` in the same format as before. Nothing else in action.yml reads these outputs.

2. The 'rustup toolchain install' step:
- **Garbled block:** part of this script had been overwritten with lines from another step's YAML. I rebuilt the override block. It takes the last toolchain after the final comma (as upstream's `${_srt_toolchain##*,}` did), splits it on whitespace, then runs `rustup override set`.
- **Leftover expression:** the stray `${{ steps.versions.outputs.rustc-version }}` expression is no longer inside that script.

3. The 'Downgrade registry access protocol when needed' step:
- **Missing env entry:** the step read `$_srt_RUSTC_VERSION` but never set it. It now gets it from `steps.versions.outputs.rustc-version` in its env block, so the downgrade check works again.

### Iteration 3

**Fixes applied:** script-injection, github-env-injection

**Notes:**

I rewrote the 'flags' step in action.yml to fix all three findings. I haven't run the action or any tests on it.

- **Expression inside the script:** the `${{ }}` expression in the downgrade line is gone. It now goes into the step's env as `_srt_downgrade`, which is either 'true' or empty, using the same expression the later install step already uses. The script turns that into ' --allow-downgrade' when it's 'true'.
- **Unquoted expansion:** the target and component lists are now read into bash arrays with `read -r -d '' -a` after commas are swapped for spaces. They still split on commas and whitespace as before, but they are no longer glob-expanded.
- **Output writes:** the targets, components and downgrade values are each passed through `printf '%s' ... | tr -d '\n\r'` and written with printf. This means an input can't add extra output entries.

Nothing inside the action reads `steps.flags.outputs`; the install step already builds its own arguments. I kept the outputs in their original ' --target X' form so the step's behaviour doesn't change.

