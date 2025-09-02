# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ripgrep (`rg`) is a line-oriented search tool written in Rust that recursively searches directories for regex patterns. It's designed to be fast, respect gitignore rules, and provide first-class support for Windows, macOS, and Linux.

## Build System & Common Commands

### Building
- `cargo build --release` - Build optimized release binary (output: `target/release/rg`)
- `cargo build --verbose --workspace` - Build all workspace crates with verbose output
- `cargo build --release --features pcre2` - Build with optional PCRE2 regex engine support
- `cargo build --target x86_64-unknown-linux-musl` - Build static MUSL binary

### Testing
- `cargo test --all` - Run all tests (unit + integration)
- `cargo test --verbose --workspace --features pcre2` - Test with PCRE2 support
- `cargo test --manifest-path crates/cli/Cargo.toml` - Test specific crate

### Code Quality
- `cargo fmt --all --check` - Check code formatting (uses `rustfmt.toml` config)
- `cargo doc --no-deps --document-private-items --workspace` - Generate documentation

### Cross-compilation
The project uses `cross` for cross-compilation to various targets including ARM, PowerPC, and s390x. See `.github/workflows/ci.yml` for complete target list.

## Architecture

### Workspace Structure
The project is organized as a Cargo workspace with multiple specialized crates:

- **`crates/core/`** - Main application entry point (`main.rs`)
  - Contains the primary CLI logic, argument parsing, and search orchestration
  - Key modules: `flags`, `haystack`, `logger`, `search`, `messages`

- **`crates/grep/`** - High-level search API facade
  - Re-exports all constituent crates for external library usage
  - Provides unified interface to ripgrep's core functionality

- **`crates/ignore/`** - Directory traversal with filtering
  - Handles `.gitignore`, `.ignore`, file type filtering
  - Provides `Walk` and `WalkBuilder` for recursive directory iteration

- **`crates/searcher/`** - Core search implementation
  - Low-level search algorithms and file processing

- **`crates/matcher/`** - Pattern matching abstractions
  - Defines trait interfaces for different regex engines

- **`crates/regex/`** - Default Rust regex engine integration
- **`crates/pcre2/`** - Optional PCRE2 regex engine (for lookaround, backreferences)
- **`crates/printer/`** - Search result formatting and output
- **`crates/globset/`** - Glob pattern matching
- **`crates/cli/`** - CLI utilities and helpers

### Binary Output
- Main binary: `target/release/rg` (defined in `[[bin]]` section)
- Build script (`build.rs`) embeds git hash and handles Windows manifest

### Key Features
- Multiple regex engines (default Rust regex + optional PCRE2)
- Parallel directory traversal using `crossbeam` and `ignore` crate
- Memory-mapped file searching for performance
- SIMD optimizations in regex engine
- Cross-platform long path support (Windows)
- Compressed file search support
- Custom file type definitions
- Configuration file support

### Testing
- Integration tests in `tests/` directory with modular organization:
  - `binary.rs` - Binary file handling
  - `feature.rs` - Main feature testing
  - `json.rs` - JSON output format
  - `multiline.rs` - Multiline search
  - `regression.rs` - Regression tests

### Build Profiles
- `release` - Standard optimized build with debug symbols
- `release-lto` - Maximum optimization with LTO, no debug info
- `deb` - Debian package build profile

### Memory Management
Uses jemalloc allocator specifically for MUSL targets (x86_64) to improve performance over musl's default allocator.