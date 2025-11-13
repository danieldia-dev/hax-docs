# Hax Library - Documentation Index

## Overview

This directory contains comprehensive documentation for the **hax** library, a powerful tool for translating Rust code into formal verification languages.

## Documentation Files

### 📚 Core Documentation

#### 1. **[README.md](./README.md)** - Complete Overview
Comprehensive introduction covering library overview, core components, type system, macros, CLI, backends, and examples.

#### 2. **[hax-lib-api.md](./hax-lib-api.md)** - API Reference
Complete API documentation for the hax-lib crate including all types, traits, macros, and functions.

#### 3. **[cli-backends.md](./cli-backends.md)** - CLI and Backends
Complete guide for using the hax CLI and all verification backends (F*, Lean, Coq, ProVerif, SSProve, EasyCrypt).

#### 4. **[examples-patterns.md](./examples-patterns.md)** - Examples and Patterns
Practical examples and design patterns from basic verification to cryptographic algorithms.

### 📚 Advanced Documentation

#### 5. **[internal-architecture.md](./internal-architecture.md)** - Internal Architecture
Deep dive into hax's implementation including compilation pipeline, frontend, engine, and backend details.

#### 6. **[macro-system-complete.md](./macro-system-complete.md)** - Macro System
Exhaustive documentation of all macros, their implementations, syntax, and usage.

#### 7. **[verification-techniques.md](./verification-techniques.md)** - Verification Techniques
Comprehensive guide to proof strategies, invariant patterns, and verification workflows.

#### 8. **[errors-troubleshooting.md](./errors-troubleshooting.md)** - Error Reference
Complete error code reference with solutions, debugging strategies, and troubleshooting guides.

## How to Use This Documentation

### For New Users
1. Start with **[README.md](./README.md)** for a complete overview
2. Review **[examples-patterns.md](./examples-patterns.md)** for practical examples
3. Refer to **[cli-backends.md](./cli-backends.md)** for installation and setup
4. Check **[errors-troubleshooting.md](./errors-troubleshooting.md)** when issues arise

### For Developers
1. Use **[hax-lib-api.md](./hax-lib-api.md)** as your primary API reference
2. Study **[macro-system-complete.md](./macro-system-complete.md)** for macro details
3. Consult **[verification-techniques.md](./verification-techniques.md)** for proof strategies
4. Reference **[internal-architecture.md](./internal-architecture.md)** for implementation details

### For Advanced Users
1. Review all backend sections in **[cli-backends.md](./cli-backends.md)**
2. Study advanced patterns in **[verification-techniques.md](./verification-techniques.md)**
3. Understand the internals via **[internal-architecture.md](./internal-architecture.md)**
4. Master the macro system with **[macro-system-complete.md](./macro-system-complete.md)**

### For Troubleshooting
1. Start with **[errors-troubleshooting.md](./errors-troubleshooting.md)** for error codes
2. Check specific backend sections in **[cli-backends.md](./cli-backends.md)**
3. Review verification failures in **[verification-techniques.md](./verification-techniques.md)**
4. Consult macro issues in **[macro-system-complete.md](./macro-system-complete.md)**

## Quick Links

### External Resources
- **GitHub Repository**: https://github.com/hacspec/hax
- **Official Website**: https://hax.cryspen.com
- **Playground**: https://hax-playground.cryspen.com
- **Zulip Chat**: https://hacspec.zulipchat.com/
- **Blog**: https://hax.cryspen.com/blog
- **Manual**: https://hax.cryspen.com/manual/

### Key Examples in Repository
- `/examples/barrett/` - Field arithmetic verification
- `/examples/chacha20/` - Stream cipher implementation
- `/examples/sha256/` - Hash function verification
- `/examples/kyber_compress/` - Lattice cryptography
- `/examples/lean_barrett/` - Lean-specific proofs
- `/examples/lean_chacha20/` - Lean panic-freedom
- `/examples/limited-order-book/` - Protocol example
- `/examples/proverif-psk/` - ProVerif protocol

## Contributing

For updates, corrections, or additions to this documentation:
1. File an issue on GitHub: https://github.com/hacspec/hax/issues
2. Join the Zulip chat: https://hacspec.zulipchat.com/
3. Contribute via pull requests with documentation improvements

## License

The hax library is licensed under Apache-2.0. See the `LICENSE` file in the repository root for details.
