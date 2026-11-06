<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v2.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v2.0.0** was hardened automatically. 10 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Step 'flags' writes values built from inputs.target and inputs.components (through _srt_targets and _srt_components) to $GITHUB_OUTPUT, and line 125 also writes the expression ${{contains(inputs.toolchain, 'nightly') && inputs.components && ...}}. None of these values goes through printf '%s' | tr -d '\n\r' first. A newline in an input would add extra output keys.

Locations:

- `action.yml:123`
- `action.yml:124`
- `action.yml:125`

### github-env-injection (severity: high)

Step 'Setting Environment Variables' writes `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV` and `echo "CARGO_BUILD_WARNINGS=$_srt_CARGO_BUILD_WARNINGS" >> $GITHUB_ENV`. The values come from inputs.rustflags and inputs.build-warnings and are not sanitized with printf '%s' | tr -d '\n\r'. A newline in either input could set any environment variable, for example LD_PRELOAD or NODE_OPTIONS.

Locations:

- `action.yml:147`
- `action.yml:150`

### github-env-injection (severity: high)

Step 'Install rustup, if needed' writes the inherited, workflow-controlled $CARGO_HOME to $GITHUB_PATH without sanitizing it: `echo "${CARGO_HOME:-$USERPROFILE/.cargo}/bin" ... >> $GITHUB_PATH` and `echo "${CARGO_HOME:-$HOME/.cargo}/bin" >> $GITHUB_PATH`. It has no printf '%s' | tr -d '\n\r' step, so a newline in the value would add extra PATH entries.

Locations:

- `action.yml:175`
- `action.yml:177`

### script-injection (severity: high)

Step 'flags' breaks both sub-rules. (a) An expression is interpolated straight into the script: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`. (b) Values that come from inputs are expanded without quotes: `for t in ${_srt_targets//,/ }; do echo -n ' --target' $t; done` and the matching loop over $c / ${_srt_components//,/ }.

Locations:

- `action.yml:123`
- `action.yml:124`
- `action.yml:125`

### script-injection (severity: high)

Step 'Setting Environment Variables' breaks sub-rule (b): `[[ ( ! -v RUSTFLAGS ) && $_srt_NEW_RUSTFLAGS != "" ]]` and `[[ ( ! -v CARGO_BUILD_WARNINGS ) && $_srt_CARGO_BUILD_WARNINGS != "" ]]` expand env vars that come from inputs without double quotes.

Locations:

- `action.yml:146`
- `action.yml:149`

### script-injection (severity: high)

Step 'Install Rust Problem Matcher' breaks sub-rule (a): `run: echo "::add-matcher::${{ github.action_path }}/rust.json"` interpolates a ${{ }} expression directly into the run script. It should use "$GITHUB_ACTION_PATH" instead.

Locations:

- `action.yml:165`

### script-injection (severity: high)

Step 'rustup toolchain install' breaks both sub-rules. (a) Line 223 interpolates expressions directly: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`. Those step outputs are built from inputs.target, inputs.components and inputs.toolchain, so this is a path for command injection. (b) Values from inputs are expanded without quotes: `[[ -n $_srt_components ]]`, `rustup component add ${_srt_components//,/ }`, `[[ -n $_srt_targets ]]`, `rustup target add ${_srt_targets//,/ }`, `${_srt_toolchain//,/ }` and `rustup override set ${_srt_toolchain//*,/ }`.

Locations:

- `action.yml:212`
- `action.yml:213`
- `action.yml:215`
- `action.yml:216`
- `action.yml:223`
- `action.yml:227`

### script-injection (severity: high)

Step 'Downgrade registry access protocol when needed' breaks sub-rule (a): `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ... ]]` interpolates a step output expression directly into the run script. The value should be passed in through an env: variable instead.

Locations:

- `action.yml:256`

### unsafe-shell (severity: high)

Step 'Install rustup, if needed' pipes remote content straight into a shell: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. The installer should be downloaded to a file, ideally checked, and then run as a separate step.

Locations:

- `action.yml:171`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index d8d755a..abc693a 100644
--- a/action.yml
+++ b/action.yml
@@ -106,6 +106,23 @@ outputs:
 runs:
   using: composite
   steps:
     # The later code uses "newer" bash features, which are not available in the ancient bash 3 (from 2014) of macOS
     - name: Unbork mac
       if: runner.os == 'macOS'
@@ -117,12 +134,30 @@ runs:
       env:
         _srt_targets: ${{inputs.target}}
         _srt_components: ${{inputs.components}}
+        _srt_downgrade: ${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}
       shell: bash
       run: |
         : construct rustup command line
-        echo "targets=$(for t in ${_srt_targets//,/ }; do echo -n ' --target' $t; done)" >> $GITHUB_OUTPUT
-        echo "components=$(for c in ${_srt_components//,/ }; do echo -n ' --component' $c; done)" >> $GITHUB_OUTPUT
-        echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT
+        _srt_targets_flags=""
+        if [[ -n "$_srt_targets" ]]; then
+          while IFS= read -r -d '' _srt_t; do
+            _srt_targets_flags="${_srt_targets_flags} --target ${_srt_t}"
+          done < <(printf '%s' "${_srt_targets//,/ }" | xargs -r printf '%s\0')
+        fi
+        _srt_safe_targets="$(printf '%s' "$_srt_targets_flags" | tr -d '\n\r')"
+        echo "targets=${_srt_safe_targets}" >> "$GITHUB_OUTPUT"
+
+        _srt_components_flags=""
+        if [[ -n "$_srt_components" ]]; then
+          while IFS= read -r -d '' _srt_c; do
+            _srt_components_flags="${_srt_components_flags} --component ${_srt_c}"
+          done < <(printf '%s' "${_srt_components//,/ }" | xargs -r printf '%s\0')
+        fi
+        _srt_safe_components="$(printf '%s' "$_srt_components_flags" | tr -d '\n\r')"
+        echo "components=${_srt_safe_components}" >> "$GITHUB_OUTPUT"
+
+        _srt_safe_downgrade="$(printf '%s' "$_srt_downgrade" | tr -d '\n\r')"
+        echo "downgrade=${_srt_safe_downgrade}" >> "$GITHUB_OUTPUT"
 
     # The environment variables always need to be set before the caching action
     - name: Setting Environment Variables
@@ -144,10 +179,12 @@ runs:
           echo "RUST_BACKTRACE=short" >> $GITHUB_ENV
         fi
         if [[ ( ! -v RUSTFLAGS ) && $_srt_NEW_RUSTFLAGS != "" ]]; then
-          echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV
+          _srt_SAFE_RUSTFLAGS="$(printf '%s' "$_srt_NEW_RUSTFLAGS" | tr -d '\n\r')"
+          echo "RUSTFLAGS=${_srt_SAFE_RUSTFLAGS}" >> $GITHUB_ENV
         fi
         if [[ ( ! -v CARGO_BUILD_WARNINGS ) && $_srt_CARGO_BUILD_WARNINGS != "" ]]; then
-          echo "CARGO_BUILD_WARNINGS=$_srt_CARGO_BUILD_WARNINGS" >> $GITHUB_ENV
+          _srt_SAFE_CARGO_BUILD_WARNINGS="$(printf '%s' "$_srt_CARGO_BUILD_WARNINGS" | tr -d '\n\r')"
+          echo "CARGO_BUILD_WARNINGS=${_srt_SAFE_CARGO_BUILD_WARNINGS}" >> $GITHUB_ENV
         fi
         # Enable faster sparse index on nightly
         # The value is ignored on stable and causes no problems
@@ -161,14 +198,19 @@ runs:
 
     - name: Install Rust Problem Matcher
       if: inputs.matcher == 'true'
+      env:
+        _srt_action_path: ${{ github.action_path }}
       shell: bash
-      run: echo "::add-matcher::${{ github.action_path }}/rust.json"
+      run: echo "::add-matcher::${_srt_action_path}/rust.json"
 
     - name: Install rustup, if needed
       shell: bash
       run: |
         if ! command -v rustup &> /dev/null ; then
-          curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y
+          _srt_rustup_init="$(mktemp)"
+          curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" -o "$_srt_rustup_init"
+          sh "$_srt_rustup_init" --default-toolchain none -y
+          rm -f "$_srt_rustup_init"
 
           # Resolve the correct CARGO_HOME path depending on OS
           if [[ "$RUNNER_OS" == "Windows" ]]; then
@@ -187,6 +229,9 @@ runs:
      
... (truncated)
```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection, unsafe-shell, prior-fix

**Notes:**

All fixes are in action.yml. I have not run the action to test them.

- **flags step:** The allow-downgrade expression now goes in through an env var (_srt_downgrade). Targets and components are split into lists with `read -r -d '' -a` after commas become spaces, which matches how upstream's unquoted loop split them. The targets, components and downgrade values have newlines and carriage returns stripped before they are written to $GITHUB_OUTPUT.
- **Setting Environment Variables step:** The conditions now quote the input variables. RUSTFLAGS and CARGO_BUILD_WARNINGS have newlines and carriage returns stripped before they are written to $GITHUB_ENV.
- **Problem matcher step:** Uses $GITHUB_ACTION_PATH instead of the inline expression.
- **Install rustup step:** The rustup installer is downloaded to a mktemp file and then run with `sh "$f" --default-toolchain none -y`; the `-s --` from the piped form is dropped. The cargo bin path (from CARGO_HOME) has newlines stripped before it is written to $GITHUB_PATH, and the Windows backslash conversion is kept.
- **rustup toolchain install step:** The three ${{ steps.flags.outputs.* }} expressions are gone. The step now builds bash arrays straight from its env vars: the toolchain list, then the --target, --component, --profile minimal, --allow-downgrade and --no-self-update flags, and the last toolchain for `rustup override set`. Lists split on commas and whitespace, as upstream's did. The component and target add commands use the arrays, and the `-n` checks are quoted.
- **Downgrade registry step:** rustc-version now comes in through an env var.

The flags step still writes its outputs, now cleaned, but the install step no longer reads them.

Not changed: the 'versions' step still writes rustc, cargo and rustup version text to $GITHUB_OUTPUT without stripping newlines. That text is command output, not an input, and the analysis did not flag it.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

In the 'Downgrade registry access protocol when needed' step, I replaced the inline ${{steps.versions.outputs.rustc-version}} in the run: script with "$_srt_rustc_version". That variable was already mapped from the same output in the step's env: block.

