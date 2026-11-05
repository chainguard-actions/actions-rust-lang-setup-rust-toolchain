<!-- markdownlint-disable -->

# Hardening Report: actions-rust-lang--setup-rust-toolchain/v1.17.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **actions-rust-lang--setup-rust-toolchain/v1.17.0** was hardened automatically. 9 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Step 'flags': (a) an expression is placed directly in the run script: `echo "downgrade=${{contains(inputs.toolchain, 'nightly') && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT`. (b) Input-derived env vars are expanded without quotes: `for t in ${_srt_targets//,/ }; do echo -n ' --target' $t; done` and the same pattern for _srt_components / $c.

Locations:

- `action.yml:119`
- `action.yml:120`
- `action.yml:121`

### script-injection (severity: high)

Step 'Install Rust Problem Matcher': (a) an expression is placed directly in the run script: `run: echo "::add-matcher::${{ github.action_path }}/rust.json"`.

Locations:

- `action.yml:157`

### script-injection (severity: high)

Step 'rustup toolchain install': (a) step outputs are placed directly in the script: `rustup toolchain install ${_srt_toolchain//,/ } ${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}} --no-self-update`. (b) Input-derived vars are expanded without quotes: `${_srt_toolchain//,/ }`, `rustup component add ${_srt_components//,/ }`, `rustup target add ${_srt_targets//,/ }`, `rustup override set ${_srt_toolchain//*,/ }`, and `[[ -n $_srt_components ]]`.

Locations:

- `action.yml:204`
- `action.yml:205`
- `action.yml:208`
- `action.yml:215`
- `action.yml:219`

### script-injection (severity: high)

Step 'Downgrade registry access protocol when needed': (a) a step output is placed directly in the script: `if [[ "${{steps.versions.outputs.rustc-version}}" =~ ^rustc\ (1\.6[67]\.|1\.68\.0-nightly) && ...`.

Locations:

- `action.yml:247`

### github-env-injection (severity: high)

Step 'flags': values built from inputs.target and inputs.components (through the env vars _srt_targets and _srt_components) are written to $GITHUB_OUTPUT without the `printf '%s' ... | tr -d '\n\r'` sanitization step: `echo "targets=$(for t in ${_srt_targets//,/ }; ...)" >> $GITHUB_OUTPUT` and the same for components.

Locations:

- `action.yml:119`
- `action.yml:120`

### github-env-injection (severity: high)

Step 'Setting Environment Variables': inputs.rustflags (through _srt_NEW_RUSTFLAGS) is written to $GITHUB_ENV without sanitization: `echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV`. A value containing a newline can inject extra environment variables.

Locations:

- `action.yml:142`

### github-env-injection (severity: high)

Step 'Install rustup, if needed': CARGO_HOME is inherited from the calling workflow and is written to $GITHUB_PATH without sanitization: `echo "${CARGO_HOME:-$USERPROFILE/.cargo}/bin" | sed ... >> $GITHUB_PATH` and `echo "${CARGO_HOME:-$HOME/.cargo}/bin" >> $GITHUB_PATH`.

Locations:

- `action.yml:167`
- `action.yml:169`

### unsafe-shell (severity: high)

A remote script is piped straight into a shell: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused -fsSL "https://sh.rustup.rs" | sh -s -- --default-toolchain none -y`. Download it to a file first, then run it as a separate step.

Locations:

- `action.yml:163`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 35554cf..119a647 100644
--- a/action.yml
+++ b/action.yml
@@ -2,11 +2,9 @@ name: Setup Rust Toolchain for GitHub CI
 description: |
   Setup specific Rust versions with caching pre-configured.
   It provides problem matchers for cargo and rustfmt issues.
-
 branding:
   icon: "play"
   color: "gray-dark"
-
 # Add option to install directly from rust-toolchain file
 # Blocked on rustup support: https://github.com/rust-lang/rustup/issues/2686
 #
@@ -80,7 +78,6 @@ inputs:
   rust-src-dir:
     description: "Specify path from root directory to the Rust source directory. By default root directory will be used."
     required: false
-
 outputs:
   rustc-version:
     description: "Version as reported by `rustc --version`"
@@ -94,7 +91,6 @@ outputs:
   cachekey:
     description: A short hash of the rustc version, appropriate for use as a cache key. "20220627a831"
     value: ${{steps.versions.outputs.cachekey}}
-
 # All local variables are using the `_srt_` prefix (for setup-rust-toolchain)
 # this is to avoid conflicts with globally existing variables, as some names are quite generic (e.g., toolchain)
 # This has caused issues in the past.
@@ -102,24 +98,49 @@ outputs:
 runs:
   using: composite
   steps:
     # The later code uses "newer" bash features, which are not available in the ancient bash 3 (from 2014) of macOS
     - name: Unbork mac
       if: runner.os == 'macOS'
       run: |
         brew install bash
       shell: bash
-
     - id: flags
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
-
+        _srt_targets_safe=$(printf '%s' "$_srt_targets" | tr -d '\n\r')
+        _srt_components_safe=$(printf '%s' "$_srt_components" | tr -d '\n\r')
+        _srt_downgrade_safe=$(printf '%s' "$_srt_downgrade" | tr -d '\n\r')
+        targets_val=""
+        IFS=',' read -ra _srt_targets_arr <<< "$_srt_targets_safe"
+        for t in "${_srt_targets_arr[@]}"; do [[ -n "$t" ]] && targets_val="$targets_val --target $t"; done
+        components_val=""
+        IFS=',' read -ra _srt_components_arr <<< "$_srt_components_safe"
+        for c in "${_srt_components_arr[@]}"; do [[ -n "$c" ]] && components_val="$components_val --component $c"; done
+        echo "targets=$targets_val" >> $GITHUB_OUTPUT
+        echo "components=$components_val" >> $GITHUB_OUTPUT
+        echo "downgrade=$_srt_downgrade_safe" >> $GITHUB_OUTPUT
     # The environment variables always need to be set before the caching action
     - name: Setting Environment Variables
       env:
@@ -139,7 +160,8 @@ runs:
           echo "RUST_BACKTRACE=short" >> $GITHUB_ENV
         fi
         if [[ ( ! -v RUSTFLAGS ) && $_srt_NEW_RUSTFLAGS != "" ]]; then
-          echo "RUSTFLAGS=$_srt_NEW_RUSTFLAGS" >> $GITHUB_ENV
+          _srt_SAFE_RUSTFLAGS=$(printf '%s' "$_srt_NEW_RUSTFLAGS" | tr -d '\n\r')
+          echo "RUSTFLAGS=$_srt_SAFE_RUSTFLAGS" >> $GITHUB_ENV
         fi
         # Enable faster sparse index on nightly
         # The value is ignored on stable and causes no problems
@@ -150,17 +172,21 @@ runs:
         if [[ ! -v CARGO_REGISTRIES_CRATES_IO_PROTOCOL ]]; then
           echo "CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse" >> $GITHUB_ENV
         fi
-
     - name: Install Rust Problem Matcher
       if: inputs.matcher == 'true'
+      env:
+        _srt_action_path: ${{ github.action_path }}
       shell: bash
-      run: echo "::add-matcher::${{ github.action_path }}/rust.json"
-
+      run: |
+        echo "::add
... (truncated)
```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell, prior-fix

**Notes:**

flags step: the downgrade expression moved to env; targets/components split on commas and whitespace into arrays (same split upstream did); outputs newline-stripped. RUSTFLAGS newline-stripped before $GITHUB_ENV. Matcher step: github.action_path moved to env. rustup installer downloaded to a mktemp file, then run with 'sh file --default-toolchain none -y' (the '-s --' is dropped). Cargo bin path newline-stripped before $GITHUB_PATH. Install step: flags outputs moved to env; toolchain/targets/components/override split into bash arrays behind non-empty guards and passed quoted. rustc-version output moved to env in the downgrade-protocol step. The edit tool mishandled several parallel edits (one reported success but did not apply, one put text inside a comment line); I repaired the comment, redid the lost edit, and checked the file.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Added double quotes around $_srt_NEW_RUSTFLAGS in the [[ ]] test of the 'Setting Environment Variables' step in action.yml.

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

In action.yml's 'versions' step, each output (rustc-version, cargo-version, rustup-version, cachekey) is now captured with `raw="$(cmd)" || true`, which keeps upstream's behavior where a failing command inside echo did not stop the step. Each value then goes through printf '%s' | tr -d '\n\r' before it is written to the quoted "$GITHUB_OUTPUT".

