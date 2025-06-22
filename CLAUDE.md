# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This repository contains three main projects:

1. **CGP (Context-Generic Programming)** - `/cgp/`: Core CGP framework and libraries
2. **Hypershell** - `/hypershell/`: Type-level DSL for shell scripting built on CGP
3. **Website** - `/contextgeneric.dev/`: Project documentation website

## Essential Commands

### CGP Development

```bash
# Navigate to CGP workspace
cd cgp

# Build all crates
cargo build --workspace

# Run all tests
cargo test --workspace

# Run specific crate tests
cargo test -p cgp-tests

# Build documentation
cargo doc --workspace --no-deps

# Check all crates
cargo check --workspace

# Format code
cargo fmt --all

# Run clippy
cargo clippy --workspace
```

### Hypershell Development

```bash
# Navigate to Hypershell workspace
cd hypershell

# Build all crates
cargo build --workspace

# Run examples
cargo run --example hello
cargo run --example http_checksum_native

# Run tests
cargo test --workspace

# Run specific example
cargo run --bin hypershell-examples --example bluesky
```

### Website Development

```bash
# Navigate to website
cd contextgeneric.dev

# Build website (requires Zola)
zola build

# Serve locally
zola serve

# Build with Nix
nix build
```

## Architecture Overview

### Context-Generic Programming (CGP)

CGP is a modular programming paradigm built around three core concepts:

- **Contexts**: Structures that encapsulate state and behavior
- **Components**: Named capabilities that contexts can have (traits)
- **Providers**: Implementations of specific components

Key architectural patterns:
- **Type-level programming**: Components wired at compile time
- **Delegation system**: `UseDelegate` and `UseContext` for dynamic dispatch
- **Preset system**: Composable, inheritable component configurations
- **Macro-driven development**: Extensive code generation for ergonomics

### Hypershell DSL

Hypershell is a type-level DSL for shell scripting that:
- Uses phantom types to represent operations at zero runtime cost
- Leverages CGP's component system for extensible execution
- Provides streaming pipelines similar to Unix pipes
- Mixes CLI commands with native Rust functions

Core DSL constructs:
- `SimpleExec<Path, Args>`: Execute commands and return output
- `StreamingExec<Path, Args>`: Execute with streaming I/O
- `Pipe<Handlers>`: Chain operations together
- `Use<Provider, Code>`: Delegate to specific providers

## Key Crates and Their Purposes

### CGP Core Crates
- `cgp-core`: Fundamental traits and abstractions
- `cgp-component`: Component system (providers, delegation)
- `cgp-macro`: Procedural macros for code generation
- `cgp-async`: Async trait support and utilities
- `cgp-error`: Error handling abstractions
- `cgp-field`: Type-safe field access
- `cgp-type`: Type-level programming utilities

### Hypershell Crates
- `hypershell`: Main crate with presets and contexts
- `hypershell-components`: Core DSL constructs and traits
- `hypershell-macro`: `hypershell!` procedural macro
- `hypershell-tokio-components`: Async process execution
- `hypershell-reqwest-components`: HTTP client integration
- `hypershell-json-components`: JSON serialization/deserialization
- `hypershell-hash-components`: Cryptographic operations
- `hypershell-examples`: Example programs and use cases

## Development Patterns

### Creating New Components

1. Define the component trait with `#[cgp_component]`
2. Create provider implementations
3. Wire components in presets using `cgp_preset!`
4. Use `delegate_components!` for type-level mappings

### Extending Hypershell

1. Create new DSL constructs as phantom types
2. Implement `Handler<Context, Code, Input>` trait
3. Add to relevant presets
4. Update macro for syntax support if needed

### Testing Strategy

- Unit tests in individual crates
- Integration tests in `cgp-tests` and `hypershell-examples`
- Example programs serve as both demos and tests
- Compile-time verification through type system

## Rust Toolchain Requirements

- **CGP**: Rust 1.85+ (stable channel)
- **Hypershell**: Rust 1.87+ (stable channel)
- Edition 2021 for CGP, Edition 2024 for Hypershell

## Important Notes

- All CGP dispatch happens at compile time for zero-cost abstractions
- Hypershell macros transform readable syntax into type-level constructs
- The preset system enables modular composition of capabilities
- Error handling uses `anyhow` for most components
- Heavy use of phantom types requires careful attention to type parameters