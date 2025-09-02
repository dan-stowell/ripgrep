# Ripgrep Bazel Migration Plan

This document outlines a step-by-step plan to migrate the ripgrep project to use Bazel for building and testing alongside the existing Cargo build system using bzlmod.

## Prerequisites

- Bazel 7.0+ (with bzlmod support)
- rules_rust 0.63.0+
- Existing Cargo workspace maintained for compatibility

## Migration Steps

### Step 1: Initialize Bazel Configuration

**Goal**: Set up basic Bazel workspace with rules_rust integration.

**Actions**:
1. Create `MODULE.bazel` file in repository root
2. Create `.bazelrc` configuration file
3. Create root `BUILD.bazel` file with workspace-level aliases

**Expected Bazel targets after completion**:
- `//...` (basic workspace scanning should work)
- No buildable targets yet

**Files created**:
```
MODULE.bazel
.bazelrc
BUILD.bazel
```

**Content of MODULE.bazel**:
```starlark
module(
    name = "ripgrep",
    version = "14.1.1",
)

bazel_dep(name = "rules_rust", version = "0.63.0")
bazel_dep(name = "platforms", version = "0.0.11")
bazel_dep(name = "bazel_skylib", version = "1.7.1")

rust = use_extension("@rules_rust//rust:extensions.bzl", "rust")
rust.toolchain(edition = "2021")
use_repo(rust, "rust_toolchains")
register_toolchains("@rust_toolchains//:all")

# External dependencies will be added in Step 2
```

---

### Step 2: Configure External Dependencies

**Goal**: Set up crate_universe to manage external dependencies from Cargo.toml files.

**Actions**:
1. Add crate_universe configuration to MODULE.bazel
2. Generate initial dependency lockfiles
3. Run `bazel mod deps` to fetch dependencies

**Expected Bazel targets after completion**:
- `@ripgrep_deps//:anyhow`
- `@ripgrep_deps//:bstr`
- `@ripgrep_deps//:grep`
- `@ripgrep_deps//:ignore`
- `@ripgrep_deps//:lexopt`
- `@ripgrep_deps//:log`
- `@ripgrep_deps//:serde_json`
- `@ripgrep_deps//:termcolor`
- `@ripgrep_deps//:textwrap`
- All transitive dependencies from external crates

**Files created/modified**:
```
MODULE.bazel (updated)
cargo-bazel-lock.json (generated)
```

**MODULE.bazel addition**:
```starlark
crate_universe_deps = use_extension("@rules_rust//crate_universe:extensions.bzl", "crate")
crate_universe_deps.from_cargo(
    name = "ripgrep_deps",
    cargo_lockfile = "//:Cargo.lock",
    manifests = [
        "//:Cargo.toml",
        "//crates/globset:Cargo.toml",
        "//crates/grep:Cargo.toml",
        "//crates/cli:Cargo.toml",
        "//crates/matcher:Cargo.toml",
        "//crates/pcre2:Cargo.toml", 
        "//crates/printer:Cargo.toml",
        "//crates/regex:Cargo.toml",
        "//crates/searcher:Cargo.toml",
        "//crates/ignore:Cargo.toml",
    ],
)
use_repo(crate_universe_deps, "ripgrep_deps")
```

---

### Step 3: Build Core Library Crates

**Goal**: Create BUILD.bazel files for all library crates in the workspace.

**Actions**:
1. Create `crates/globset/BUILD.bazel`
2. Create `crates/matcher/BUILD.bazel`
3. Create `crates/regex/BUILD.bazel`
4. Create `crates/searcher/BUILD.bazel`
5. Create `crates/printer/BUILD.bazel`
6. Create `crates/cli/BUILD.bazel`
7. Create `crates/ignore/BUILD.bazel`

**Expected Bazel targets after completion**:
- `//crates/globset:globset`
- `//crates/matcher:matcher` 
- `//crates/regex:regex`
- `//crates/searcher:searcher`
- `//crates/printer:printer`
- `//crates/cli:cli`
- `//crates/ignore:ignore`

**Expected Bazel tests after completion**:
- `//crates/globset:globset_test`
- `//crates/matcher:matcher_test`
- `//crates/regex:regex_test`
- `//crates/searcher:searcher_test`
- `//crates/printer:printer_test`
- `//crates/cli:cli_test`
- `//crates/ignore:ignore_test`

**Example BUILD.bazel for crates/globset**:
```starlark
load("@rules_rust//rust:defs.bzl", "rust_library", "rust_test")

rust_library(
    name = "globset",
    srcs = glob(["src/**/*.rs"]),
    crate_root = "src/lib.rs",
    edition = "2021",
    visibility = ["//visibility:public"],
    deps = [
        "@ripgrep_deps//:aho_corasick",
        "@ripgrep_deps//:bstr",
        "@ripgrep_deps//:log",
        "@ripgrep_deps//:regex",
        "@ripgrep_deps//:serde",
    ],
)

rust_test(
    name = "globset_test", 
    crate = ":globset",
    edition = "2021",
)
```

---

### Step 4: Build PCRE2 Optional Crate

**Goal**: Handle conditional compilation for the optional PCRE2 feature.

**Actions**:
1. Create `crates/pcre2/BUILD.bazel` with conditional compilation
2. Configure PCRE2 system library dependencies

**Expected Bazel targets after completion**:
- `//crates/pcre2:pcre2` (conditional on pcre2 feature)

**Expected Bazel tests after completion**:
- `//crates/pcre2:pcre2_test` (conditional on pcre2 feature)

**Example BUILD.bazel for crates/pcre2**:
```starlark
load("@rules_rust//rust:defs.bzl", "rust_library", "rust_test")

rust_library(
    name = "pcre2",
    srcs = glob(["src/**/*.rs"]),
    crate_root = "src/lib.rs", 
    edition = "2021",
    visibility = ["//visibility:public"],
    deps = [
        "@ripgrep_deps//:libc",
        "@ripgrep_deps//:log",
        "@ripgrep_deps//:pcre2_sys",
        "//crates/matcher",
    ],
)

rust_test(
    name = "pcre2_test",
    crate = ":pcre2",
    edition = "2021",
)
```

---

### Step 5: Build Grep Facade Crate

**Goal**: Create the high-level grep crate that re-exports constituent crates.

**Actions**:
1. Create `crates/grep/BUILD.bazel`
2. Configure optional PCRE2 feature dependency

**Expected Bazel targets after completion**:
- `//crates/grep:grep`
- `//crates/grep:grep_pcre2` (with pcre2 feature enabled)

**Expected Bazel tests after completion**:
- `//crates/grep:grep_test`

**Example BUILD.bazel for crates/grep**:
```starlark
load("@rules_rust//rust:defs.bzl", "rust_library", "rust_test")

rust_library(
    name = "grep",
    srcs = glob(["src/**/*.rs"]),
    crate_root = "src/lib.rs",
    edition = "2021", 
    visibility = ["//visibility:public"],
    deps = [
        "//crates/cli",
        "//crates/matcher", 
        "//crates/printer",
        "//crates/regex",
        "//crates/searcher",
    ],
)

rust_library(
    name = "grep_pcre2",
    srcs = glob(["src/**/*.rs"]),
    crate_root = "src/lib.rs",
    crate_features = ["pcre2"],
    edition = "2021",
    visibility = ["//visibility:public"], 
    deps = [
        "//crates/cli",
        "//crates/matcher",
        "//crates/pcre2",
        "//crates/printer",
        "//crates/regex",
        "//crates/searcher",
    ],
)

rust_test(
    name = "grep_test",
    crate = ":grep",
    edition = "2021",
)
```

---

### Step 6: Build Main Binary with Build Script

**Goal**: Create the main ripgrep binary with build script support.

**Actions**:
1. Create `crates/core/BUILD.bazel`
2. Handle `build.rs` script for git hash and Windows manifest
3. Create genrule for build script functionality

**Expected Bazel targets after completion**:
- `//crates/core:rg` (main binary)
- `//crates/core:rg_pcre2` (binary with PCRE2 support)
- `//crates/core:build_script_gen` (build script outputs)

**Example BUILD.bazel for crates/core**:
```starlark
load("@rules_rust//rust:defs.bzl", "rust_binary")

# Handle build.rs functionality with genrule
genrule(
    name = "git_hash_gen",
    outs = ["git_hash.txt"],
    cmd = "git rev-parse --short=10 HEAD > $@",
    stamp = 1,
)

rust_binary(
    name = "rg",
    srcs = glob([
        "src/**/*.rs",
        "*.rs",
    ]),
    crate_root = "main.rs",
    edition = "2021",
    env = {
        "RIPGREP_BUILD_GIT_HASH": "$(cat $(location :git_hash_gen))",
    },
    visibility = ["//visibility:public"],
    deps = [
        "@ripgrep_deps//:anyhow",
        "@ripgrep_deps//:bstr", 
        "@ripgrep_deps//:lexopt",
        "@ripgrep_deps//:log",
        "@ripgrep_deps//:serde_json",
        "@ripgrep_deps//:termcolor",
        "@ripgrep_deps//:textwrap",
        "//crates/grep",
        "//crates/ignore",
        # Conditional jemalloc for musl
    ] + select({
        "@platforms//os:linux": ["@ripgrep_deps//:jemallocator"],
        "//conditions:default": [],
    }),
    data = [":git_hash_gen"],
)

rust_binary(
    name = "rg_pcre2",
    srcs = glob([
        "src/**/*.rs", 
        "*.rs",
    ]),
    crate_root = "main.rs",
    crate_features = ["pcre2"],
    edition = "2021",
    env = {
        "RIPGREP_BUILD_GIT_HASH": "$(cat $(location :git_hash_gen))", 
    },
    visibility = ["//visibility:public"],
    deps = [
        "@ripgrep_deps//:anyhow",
        "@ripgrep_deps//:bstr",
        "@ripgrep_deps//:lexopt", 
        "@ripgrep_deps//:log",
        "@ripgrep_deps//:serde_json",
        "@ripgrep_deps//:termcolor", 
        "@ripgrep_deps//:textwrap",
        "//crates/grep:grep_pcre2",
        "//crates/ignore",
    ] + select({
        "@platforms//os:linux": ["@ripgrep_deps//:jemallocator"],
        "//conditions:default": [],
    }),
    data = [":git_hash_gen"],
)
```

---

### Step 7: Integration Tests

**Goal**: Convert integration tests to Bazel test targets.

**Actions**:
1. Create `tests/BUILD.bazel`
2. Set up integration test binary as rust_test target
3. Configure test data and dependencies

**Expected Bazel targets after completion**:
- `//tests:integration_test`
- Individual test targets for test modules

**Expected Bazel tests after completion**:
- `//tests:integration_test`
- `//tests:binary_test`
- `//tests:feature_test`
- `//tests:json_test`
- `//tests:multiline_test`
- `//tests:regression_test`

**Example BUILD.bazel for tests**:
```starlark
load("@rules_rust//rust:defs.bzl", "rust_test")

rust_test(
    name = "integration_test",
    srcs = glob(["**/*.rs"]),
    crate_root = "tests.rs",
    edition = "2021",
    deps = [
        "@ripgrep_deps//:serde",
        "@ripgrep_deps//:serde_derive", 
        "@ripgrep_deps//:walkdir",
        "//crates/core:rg",
    ],
    data = glob(["data/**/*"]),
)

# Individual test modules
rust_test(
    name = "binary_test",
    srcs = [
        "tests.rs",
        "binary.rs", 
        "macros.rs",
        "util.rs",
        "hay.rs",
    ],
    crate_root = "tests.rs",
    edition = "2021",
    deps = [
        "@ripgrep_deps//:serde",
        "@ripgrep_deps//:serde_derive",
        "@ripgrep_deps//:walkdir", 
        "//crates/core:rg",
    ],
    data = glob(["data/**/*"]),
)
```

---

### Step 8: Code Quality Tools Integration

**Goal**: Add support for rustfmt, clippy, and documentation generation.

**Actions**:
1. Add rustfmt_test targets to all BUILD.bazel files
2. Add rust_clippy support
3. Add rust_doc targets for documentation

**Expected Bazel targets after completion**:
- `//crates/globset:globset_rustfmt_test`
- `//crates/globset:globset_clippy` 
- `//crates/globset:globset_doc`
- Similar targets for all other crates
- `//crates/core:rg_rustfmt_test`
- `//crates/core:rg_clippy`

**Expected Bazel tests after completion**:
All formatting and linting tests from code quality tools.

**Example additions to BUILD.bazel files**:
```starlark
load("@rules_rust//rust:defs.bzl", "rust_library", "rust_test", "rust_doc", "rust_clippy")
load("@rules_rust//rust/private:rustfmt.bzl", "rustfmt_test")

# Add to each crate's BUILD.bazel:
rustfmt_test(
    name = "globset_rustfmt_test",
    targets = [":globset"],
)

rust_clippy(
    name = "globset_clippy",
    deps = [":globset"],
)

rust_doc(
    name = "globset_doc", 
    crate = ":globset",
)
```

---

### Step 9: Cross-Compilation Support

**Goal**: Configure cross-compilation targets to match Cargo build capabilities.

**Actions**:
1. Add platform-specific configurations to .bazelrc
2. Configure cross-compilation toolchains
3. Create platform-specific binary targets

**Expected Bazel targets after completion**:
- `//crates/core:rg_linux_musl`
- `//crates/core:rg_x86`
- `//crates/core:rg_aarch64`
- `//crates/core:rg_armv7`
- `//crates/core:rg_powerpc64`
- `//crates/core:rg_s390x`
- `//crates/core:rg_windows`
- `//crates/core:rg_macos`

**Example .bazelrc additions**:
```
# Cross-compilation configurations
build:musl --platforms=@rules_rust//rust/platform:linux-x86_64-musl
build:x86 --platforms=@rules_rust//rust/platform:linux-i686
build:aarch64 --platforms=@rules_rust//rust/platform:linux-aarch64
build:armv7 --platforms=@rules_rust//rust/platform:linux-armv7
build:windows --platforms=@rules_rust//rust/platform:windows-x86_64
build:macos --platforms=@rules_rust//rust/platform:macos-x86_64
```

---

### Step 10: Root Workspace Aliases and Final Integration

**Goal**: Create convenient aliases and verify complete build functionality.

**Actions**:
1. Update root BUILD.bazel with convenient aliases
2. Create comprehensive test suite
3. Verify all targets build and test successfully
4. Document usage patterns

**Expected Bazel targets after completion**:
- `//:rg` (alias to main binary)
- `//:rg_pcre2` (alias to PCRE2 binary)
- `//:test_all` (runs all tests)
- `//:clippy_all` (runs all clippy checks)
- `//:format_all` (runs all format tests)

**Expected Bazel tests after completion**:
All tests should pass:
- `bazel test //...`
- `bazel test //:test_all`

**Final root BUILD.bazel**:
```starlark
load("@rules_rust//rust:defs.bzl", "rust_test_suite")

# Convenient aliases
alias(
    name = "rg",
    actual = "//crates/core:rg",
    visibility = ["//visibility:public"],
)

alias(
    name = "rg_pcre2", 
    actual = "//crates/core:rg_pcre2",
    visibility = ["//visibility:public"],
)

# Test suite aggregation
test_suite(
    name = "test_all",
    tests = [
        "//crates/globset:globset_test",
        "//crates/matcher:matcher_test",
        "//crates/regex:regex_test", 
        "//crates/searcher:searcher_test",
        "//crates/printer:printer_test",
        "//crates/cli:cli_test",
        "//crates/ignore:ignore_test",
        "//crates/pcre2:pcre2_test",
        "//crates/grep:grep_test", 
        "//tests:integration_test",
    ],
)

test_suite(
    name = "format_all",
    tests = [
        "//crates/globset:globset_rustfmt_test",
        "//crates/matcher:matcher_rustfmt_test",
        "//crates/regex:regex_rustfmt_test",
        "//crates/searcher:searcher_rustfmt_test", 
        "//crates/printer:printer_rustfmt_test",
        "//crates/cli:cli_rustfmt_test",
        "//crates/ignore:ignore_rustfmt_test",
        "//crates/pcre2:pcre2_rustfmt_test",
        "//crates/grep:grep_rustfmt_test",
        "//crates/core:rg_rustfmt_test",
    ],
)
```

---

## Final Verification Commands

After completing all steps, the following commands should work:

```bash
# Build main binary
bazel build //:rg

# Build with PCRE2
bazel build //:rg_pcre2  

# Run all tests
bazel test //...

# Run specific test suites
bazel test //:test_all
bazel test //:format_all

# Cross-compile examples
bazel build //crates/core:rg --config=musl
bazel build //crates/core:rg --config=windows

# Generate documentation
bazel build //crates/grep:grep_doc

# Run clippy
bazel build //crates/core:rg_clippy
```

## Integration with CI/CD

The existing GitHub Actions CI should be updated to run Bazel builds in parallel with Cargo builds:

```yaml
- name: Build with Bazel
  run: bazel build //...

- name: Test with Bazel  
  run: bazel test //...

- name: Cross-compile with Bazel
  run: |
    bazel build //crates/core:rg --config=musl
    bazel build //crates/core:rg --config=windows
```

This migration plan maintains full compatibility with the existing Cargo build while providing a parallel Bazel build system with enhanced cross-compilation and testing capabilities.