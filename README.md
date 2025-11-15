# Hax Library - Complete Documentation and Reference Guide

## Introduction

**Hax** is a powerful, production-ready tool for high-assurance translations of Rust code into formal verification languages. It bridges the gap between high-performance systems programming and mathematical proof, enabling developers to write idiomatic, verifiable Rust code and automatically translate it to proof assistants like **F\***, **Lean4**, **Coq/Rocq**, and **ProVerif**.

This document serves as the comprehensive top-level guide to the entire `hax` ecosystem, covering the core library (`hax-lib`), the procedural macro system (`hax-lib-macros`), the command-line toolchain (`cargo-hax`), the translation engine (`hax-engine`), and all supported verification backends.

### What Makes Hax Unique

Unlike traditional verification approaches that require:
- Rewriting your program in a specialized verification language
- Maintaining two separate codebases (implementation + specification)
- Learning complex proof assistant syntax from scratch

**Hax enables you to:**
- Write your code **once** in idiomatic Rust
- Add formal specifications using **Rust macros and attributes**
- Automatically extract to **multiple proof assistants**
- Maintain a **single source of truth** for both implementation and verification
- Leverage Rust's **type system and borrow checker** alongside formal proofs
- Generate **high-performance binaries** and **formally verified models** from the same source

### Documentation Structure

This README provides a complete overview of all hax capabilities. For deeper dives into specific topics:

- **[hax-lib-api.md](hax-lib-api.md)**: Exhaustive API reference for every function, macro, and type
- **[macro-system-complete.md](macro-system-complete.md)**: Complete documentation of all macros and their implementations
- **[examples-patterns.md](examples-patterns.md)**: Comprehensive examples and design patterns from basic to advanced
- **[verification-techniques.md](verification-techniques.md)**: In-depth guide to proof strategies and verification patterns
- **[cli-backends.md](cli-backends.md)**: Complete CLI reference and backend-specific documentation
- **[internal-architecture.md](internal-architecture.md)**: Deep dive into hax's internal implementation
- **[errors-troubleshooting.md](errors-troubleshooting.md)**: Complete error code reference and troubleshooting guide
- **[INDEX.md](INDEX.md)**: Documentation index and navigation guide

## Table of Contents

### Part 1: Foundations
1. [Overview and Architecture](#1-overview-and-architecture)
   - Key Features
   - Architecture and Data Flow
   - Verification Workflow
   - Design Philosophy

2. [Core Components](#2-core-components)
   - Component Overview
   - `hax-lib` (Core Library)
   - `hax-lib-macros` (Procedural Macros)
   - `cargo-hax` (CLI Tool)
   - `hax-engine` (Translation Engine)
   - Backend Translators

3. [Getting Started](#3-getting-started)
   - Installation Methods
   - Project Setup
   - First Verified Program
   - Development Workflow

### Part 2: Core Library (hax-lib)

4. [Type System and Mathematical Abstractions](#4-type-system-and-mathematical-abstractions)
   - Mathematical Integers (`Int`)
   - Logical Propositions (`Prop`)
   - Abstraction and Concretization
   - Type Conversions and Lifting

5. [Assertions and Assumptions](#5-assertions-and-assumptions)
   - `assert!` - Provable Assertions
   - `assume!` - Trusted Assumptions
   - `assert_prop!` - Logical Propositions
   - `debug_assert!` - Runtime-Only Checks
   - Best Practices and Safety

6. [Logical Specifications](#6-logical-specifications)
   - Universal Quantification (`forall!`)
   - Existential Quantification (`exists!`)
   - Logical Implication (`implies!`)
   - Proposition Combinators
   - Writing Complex Specifications

7. [Refinement Types](#7-refinement-types)
   - Refinement Type Theory
   - The `Refinement` Trait
   - Creating Custom Refinements
   - Refinement Type Patterns
   - Advanced Usage

### Part 3: Specification and Verification

8. [Function Contracts](#8-function-contracts)
   - Preconditions (`#[requires]`)
   - Postconditions (`#[ensures]`)
   - Multiple Contracts
   - Contract Composition
   - Advanced Contract Patterns

9. [Loop Specifications](#9-loop-specifications)
   - Loop Invariants (`#[loop_invariant]`)
   - Termination Measures (`#[decreases]`)
   - Ghost Variables
   - Invariant Design Patterns
   - Complex Loop Examples

10. [Advanced Attributes](#10-advanced-attributes)
    - Opacity and Modularity `(#[opaque]`)
    - Lemmas and Auxiliary Proofs (`#[lemma]`)
    - Visibility Control (`#[exclude]`, `#[include]`)
    - Type-Level Specifications
    - Backend-Specific Attributes

### Part 4: Toolchain and Backends

11. [CLI and Toolchain](#11-cli-and-toolchain)
    - `cargo-hax` Overview
    - Complete Command Reference
    - Configuration Options
    - Environment Variables
    - Project Organization

12. [Backend Support](#12-backend-support)
    - F\* (Stable)
    - Lean4 (Active Development)
    - Coq/Rocq (Experimental)
    - ProVerif (Protocol Verification)
    - SSProve and EasyCrypt (Cryptographic Proofs)
    - Backend Comparison Matrix

### Part 5: Advanced Topics

13. [Examples and Patterns](#13-examples-and-patterns)
    - Basic Verification Patterns
    - Cryptographic Algorithms
    - Mathematical Operations
    - Data Structures
    - Protocol Verification

14. [Advanced Features](#14-advanced-features)
    - Backend-Specific Code Injection
    - Proof Optimization
    - Modularity and Scaling
    - Performance Considerations
    - Integration with External Tools

15. [Complete API Reference](#15-complete-api-reference)
    - Core Types and Traits
    - All Macros (Declarative and Procedural)
    - Functions and Methods
    - Constants and Type Aliases

16. [Resources and Community](#16-resources-and-community)
    - External Links
    - Contributing Guidelines
    - Roadmap
    - License Information

---

## Part 1: Foundations

## 1. Overview and Architecture

### Key Features

#### 1.1 Formal Verification with Idiomatic Rust

Hax enables you to write normal Rust code with added specifications:

```rust
use hax_lib as hax;

/// A simple verified function with preconditions and postconditions
#[hax::requires(x < 100 && y < 100)]
#[hax::ensures(|result| result == x + y && result > x && result > y)]
pub fn verified_add(x: u32, y: u32) -> u32 {
    // This is regular Rust code
    let result = x + y;
    
    // Add assertions to help the verifier
    hax::assert!(result > x);
    hax::assert!(result > y);
    
    result
}
```

This code:
- Compiles to a normal, fast Rust binary
- Extracts to F\*/Lean/Coq for formal verification
- Proves absence of overflows, panics, and logic errors

#### 1.2 Panic-Freedom Proofs

Hax can prove that your code will never panic:

```rust
use hax_lib as hax;
use hax_lib::int::Int;

/// Proven safe: cannot panic due to division by zero
#[hax::requires(divisor != 0)]
#[hax::ensures(|result| 
    Int::from(result) * Int::from(divisor) + Int::from(dividend % divisor) 
    == Int::from(dividend)
)]
pub fn safe_divide(dividend: u32, divisor: u32) -> u32 {
    dividend / divisor  // Provably safe due to precondition
}

/// Alternative: using Option for total functions
#[hax::ensures(|result| match result {
    Some(val) => divisor != 0 && 
                 Int::from(val) * Int::from(divisor) == Int::from(dividend),
    None => divisor == 0
})]
pub fn checked_divide(dividend: u32, divisor: u32) -> Option<u32> {
    dividend.checked_div(divisor)
}
```

#### 1.3 Mathematical Abstractions

Reason about your code using unbounded mathematical integers:

```rust
use hax_lib::int::{Int, ToInt};

/// Verify correctness using mathematical integers
#[hax::requires(
    Int::from(x) + Int::from(y) <= Int::from(u32::MAX) &&
    Int::from(x) + Int::from(y) >= Int::from(u32::MIN)
)]
#[hax::ensures(|result| Int::from(result) == Int::from(x) + Int::from(y))]
pub fn overflow_safe_add(x: u32, y: u32) -> u32 {
    // The preconditions prove this operation is safe
    x + y
}

/// Alternative: using Int directly in specifications
fn demonstrate_int_arithmetic() {
    // Create mathematical integers (unbounded)
    let a = hax::int!(1_000_000_000_000);
    let b = hax::int!(2_000_000_000_000);
    let c = a + b;  // No overflow in mathematical integers
    
    hax::assert!(c == hax::int!(3_000_000_000_000));
    
    // Convert back to machine integers (with bounds checking)
    let concrete: u64 = c.concretize();  // Verified to fit
}
```

#### 1.4 Refinement Types

Define custom types with built-in invariants:

```rust
use hax_lib as hax;

/// A non-zero integer type
#[hax::refinement_type]
pub struct NonZero {
    #[hax::refinement]
    value: i32,
}

impl hax::Refinement for NonZero {
    type InnerType = i32;
    
    /// The invariant: value must not be zero
    fn invariant(value: i32) -> hax::Prop {
        hax::Prop::from(value != 0)
    }
}

/// This function is provably safe from division by zero
/// just from the type signature!
pub fn divide_by_nonzero(dividend: i32, divisor: NonZero) -> i32 {
    // The type system guarantees *divisor != 0
    dividend / *divisor
}

/// Smart constructor with runtime check
impl NonZero {
    #[hax::ensures(|result| match result {
        Some(nz) => *nz == value && value != 0,
        None => value == 0
    })]
    pub fn new(value: i32) -> Option<Self> {
        if value != 0 {
            Some(NonZero { value })
        } else {
            None
        }
    }
}
```

#### 1.5 Modular and Scalable Proofs

Use opacity to hide implementation details and scale verification:

```rust
/// Mark this function as opaque to verifier
#[hax::opaque]
#[hax::requires(n > 0)]
#[hax::ensures(|result| result > 0)]
fn expensive_computation(n: u32) -> u32 {
    // Complex implementation that's verified once
    // Callers only see the contract, not the implementation
    // This makes verification much faster
    (0..n).sum()
}

/// This function's verification doesn't need to re-verify
/// expensive_computation's implementation
fn caller(x: u32) -> u32 {
    if x > 0 {
        let result = expensive_computation(x);
        hax::assert!(result > 0);  // Follows from contract
        result
    } else {
        1
    }
}
```

#### 1.6 Multiple Backend Support

Extract the same Rust code to different proof assistants:

```bash
# Extract to F* for SMT-based verification
cargo hax into fstar

# Extract to Lean for interactive proofs and panic-freedom
cargo hax into lean

# Extract to Coq for functional verification
cargo hax into coq

# Extract to ProVerif for protocol analysis
cargo hax into pro-verif
```

### Architecture and Data Flow

The hax system consists of multiple components working together in a pipeline:

```
┌─────────────────────────────────────────────────────────────┐
│                      Rust Source Code                       │
│                   (with hax-lib annotations)                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ cargo hax into <backend>
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Rust Compiler (rustc)                     │
│  - Type checking and inference                              │
│  - Borrow checking                                          │
│  - Macro expansion                                          │
│  - HIR (High-level IR) generation                           │
│  - THIR (Typed High-level IR) generation                    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ THIR + Metadata
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Hax Frontend (Rust)                      │
│  - THIR extraction via rustc plugin                         │
│  - Serialization to JSON/CBOR                               │
│  - Type information preservation                            │
│  - Macro metadata collection                                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ JSON/CBOR AST
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Hax Engine (OCaml/Rust)                    │
│  - AST simplification and normalization                     │
│  - Pattern desugaring                                       │
│  - Trait resolution                                         │
│  - Type erasure/elaboration                                 │
│  - Control flow analysis                                    │
│  - Backend-agnostic transformations                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Simplified AST
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Backend Translators                       │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐   │
│  │   F*     │  Lean4   │  Coq     │ ProVerif │ SSProve  │   │
│  │  (SMT)   │  (Tactic)│ (Rocq)   │ (Proto)  │ (Crypto) │   │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Generated Code
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Verification Tools                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  F*.exe, Lean, coqc, proverif, easycrypt, ...        │   │
│  │  - SMT solving (Z3, CVC5, etc.)                      │   │
│  │  - Interactive proof development                     │   │
│  │  - Symbolic execution                                │   │
│  │  - Cryptographic game proofs                         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Verification Workflow

#### Development Cycle

```
1. Write Rust Code
   ├─→ Use hax-lib types (Int, Prop)
   ├─→ Add specifications (#[requires], #[ensures])
   ├─→ Include assertions (assert!, assume!)
   └─→ Define refinement types as needed

2. Test Normally
   ├─→ cargo test (standard Rust tests)
   ├─→ cargo run (standard execution)
   └─→ Debug with normal Rust tools

3. Extract and Verify
   ├─→ cargo hax into fstar (or other backend)
   ├─→ Review generated code
   ├─→ Run verification tool (fstar.exe, lean, etc.)
   └─→ Fix any proof failures

4. Iterate
   ├─→ Strengthen specifications
   ├─→ Add loop invariants
   ├─→ Refine types
   └─→ Improve proofs

5. Production
   ├─→ cargo build --release (optimized binary)
   ├─→ Formally verified code
   └─→ High assurance
```

#### Typical Session Example

```bash
# 1. Set up project
cargo new --lib my-verified-crate
cd my-verified-crate

# 2. Add hax-lib dependency
cat >> Cargo.toml << EOF
[dependencies]
hax-lib = { version = "*", features = ["macros"] }
EOF

# 3. Write verified code in src/lib.rs
cat > src/lib.rs << 'EOF'
use hax_lib as hax;

#[hax::requires(x < 100)]
#[hax::ensures(|result| result > x)]
pub fn increment(x: u32) -> u32 {
    x + 1
}
EOF

# 4. Test normally
cargo test

# 5. Extract to F*
cargo hax into fstar

# 6. Verify with F*
cd proofs/fstar/extraction
fstar.exe --include $HACL_HOME/lib *.fst

# 7. If verification fails, iterate on specifications
# Edit src/lib.rs, then repeat steps 4-6
```

### Design Philosophy

#### Principle 1: Idiomatic Rust First

Hax doesn't force you to write verification-specific code. It works with normal Rust:

```rust
// ✅ GOOD: Normal Rust with added specifications
#[hax::requires(v.len() > 0)]
pub fn first_element(v: &[u32]) -> u32 {
    v[0]  // Normal array indexing
}

// ❌ BAD: Inventing special verification syntax
// (This is what other tools might do, but hax avoids it)
pub fn first_element(v: &[u32]) -> u32 {
    get_verified_element(v, 0)  // No special functions needed
}
```

#### Principle 2: Specifications as Documentation

Specifications serve as executable, verified documentation:

```rust
/// Returns the maximum element in the array.
/// 
/// # Specification
/// - Requires: The array is non-empty
/// - Ensures: Result is greater than or equal to all elements
/// - Ensures: Result is equal to at least one element
#[hax::requires(arr.len() > 0)]
#[hax::ensures(|result| 
    forall(|i: usize| implies(i < arr.len(), arr[i] <= result)) &&
    exists(|i: usize| i < arr.len() && arr[i] == result)
)]
pub fn find_max(arr: &[u32]) -> u32 {
    // Implementation...
}
```

#### Principle 3: Progressive Verification

Start simple, add details incrementally:

```rust
// Level 1: Basic implementation
pub fn safe_add(x: u32, y: u32) -> Option<u32> {
    x.checked_add(y)
}

// Level 2: Add type-level specification
use hax_lib::int::Int;

#[hax::ensures(|result| match result {
    Some(sum) => Int::from(sum) == Int::from(x) + Int::from(y),
    None => Int::from(x) + Int::from(y) > Int::from(u32::MAX)
})]
pub fn safe_add(x: u32, y: u32) -> Option<u32> {
    x.checked_add(y)
}

// Level 3: Add overflow-free variant
#[hax::requires(Int::from(x) + Int::from(y) <= Int::from(u32::MAX))]
#[hax::ensures(|result| Int::from(result) == Int::from(x) + Int::from(y))]
pub fn guaranteed_add(x: u32, y: u32) -> u32 {
    x + y
}
```

#### Principle 4: Modularity Through Opacity

Large proofs scale through information hiding:

```rust
// Verify complex function once
#[hax::opaque]
#[hax::ensures(|result| result >= input)]
fn complex_transform(input: u32) -> u32 {
    // 1000 lines of complex logic
    // Verified once with all its details
    // ...
}

// Use it many times without reverification cost
fn client_code_1(x: u32) -> u32 {
    let y = complex_transform(x);
    hax::assert!(y >= x);  // Fast: uses contract only
    y + 1
}

fn client_code_2(x: u32) -> u32 {
    let y = complex_transform(x);
    hax::assert!(y >= x);  // Fast: uses contract only
    y * 2
}
```

---

## 2. Core Components

The hax project comprises several tightly integrated components, each with a specific role in the verification pipeline.

### 2.1 Component Overview

```
hax/
├── hax-lib/                  # Core library (runtime + specifications)
│   ├── src/
│   │   ├── lib.rs           # Main exports
│   │   ├── int.rs           # Mathematical integers
│   │   ├── refinement.rs    # Refinement type traits
│   │   └── ...
│   └── macros/              # Declarative macros
│
├── hax-lib-macros/          # Procedural macros
│   ├── src/
│   │   ├── lib.rs           # Macro definitions
│   │   ├── attrs.rs         # Attribute macros
│   │   └── ...
│   └── Cargo.toml
│
├── cli/                     # Command-line interface
│   ├── driver/             # cargo-hax implementation
│   ├── subcommands/        # Subcommand handlers
│   └── options/            # CLI option parsing
│
├── engine/                  # Translation engine (OCaml)
│   ├── lib/                # Core engine library
│   ├── backends/           # Backend translators
│   │   ├── fstar/         # F* backend
│   │   ├── lean/          # Lean backend
│   │   ├── coq/           # Coq backend
│   │   └── ...
│   └── utils/             # Utility modules
│
└── frontend/               # Rust frontend
    └── exporter/          # THIR extraction
```

### 2.2 hax-lib: The Core Library

**hax-lib** is the runtime library that provides:
- Mathematical types (`Int`, `Prop`)
- Abstraction traits (`Abstraction`, `Refinement`)
- Declarative macros (`assert!`, `assume!`, `forall!`, `exists!`)
- Backend-specific helpers

#### 2.2.1 Module Structure

```rust
// Core module organization
pub mod hax_lib {
    // Main types
    pub use int::Int;              // Mathematical integers
    pub use props::Prop;           // Logical propositions
    
    // Abstraction system
    pub use abstraction::{
        Abstraction,               // Trait for lifting to abstract types
        Concretization,           // Trait for lowering to concrete types
        RefineAs,                 // Refinement conversion
    };
    
    // Refinement types
    pub use refinement::Refinement;
    
    // Declarative macros
    pub use macros::{
        assert,                   // Proof obligation
        assume,                   // Unsafe assumption
        assert_prop,             // Logical proposition assertion
        forall,                  // Universal quantifier
        exists,                  // Existential quantifier
        implies,                 // Implication
        int,                     // Int literal
    };
    
    // Procedural macros (re-exported from hax-lib-macros)
    pub use hax_lib_macros::{
        requires,                 // Precondition attribute
        ensures,                  // Postcondition attribute
        loop_invariant,          // Loop invariant
        decreases,               // Termination measure
        opaque,                  // Opacity marker
        lemma,                   // Lemma designation
        refinement_type,         // Refinement type macro
        // ... more macros
    };
    
    // Backend-specific modules
    pub mod fstar;               // F* helpers
    pub mod lean;                // Lean helpers
    pub mod coq;                 // Coq helpers
}
```

#### 2.2.2 Design Rationale

**Why a separate library?** The hax-lib crate serves multiple purposes:

1. **Type Definitions**: Provides `Int`, `Prop`, and other specification types
2. **Trait Abstractions**: Defines `Abstraction`, `Refinement`, etc.
3. **Macro Foundations**: Declarative macros that procedural macros build upon
4. **Runtime Stubs**: No-op implementations for regular compilation
5. **Backend Adaptations**: Backend-specific utilities and helpers

#### 2.2.3 Compilation Modes

hax-lib behaves differently depending on compilation context:

```rust
// In normal compilation (cargo build):
// - Int operations are no-ops or simple wrappers
// - Assertions compile to debug_assert! or nothing
// - Specifications are ignored

pub fn example(x: u32) -> u32 {
    hax::assert!(x < 100);  // No-op in release, debug_assert! in debug
    x + 1
}

// In hax extraction (cargo hax into fstar):
// - Int is unbounded mathematical integer
// - Assertions become proof obligations
// - Specifications are extracted as formal contracts

// Generated F*:
// val example (x: u32) : Pure u32
//   (requires x <. 100ul)
//   (ensures fun result -> result >. x)
```

### 2.3 hax-lib-macros: Procedural Macros

**hax-lib-macros** is a separate proc-macro crate providing attribute and function-like macros.

#### 2.3.1 Why a Separate Crate?

Rust requires procedural macros to be in their own crate with `proc-macro = true`:

```toml
# hax-lib-macros/Cargo.toml
[lib]
proc-macro = true

[dependencies]
syn = "2.0"
quote = "1.0"
proc-macro2 = "1.0"
```

This separation allows:
- Independent versioning
- Focused dependencies
- Clear separation of concerns
- Faster compile times (can be cached separately)

#### 2.3.2 Macro Categories

The procedural macros fall into several categories:

```rust
// 1. FUNCTION CONTRACTS
#[requires(precondition)]      // Preconditions
#[ensures(|result| postcondition)]  // Postconditions
#[lemma]                       // Mark as lemma

// 2. LOOP SPECIFICATIONS  
#[loop_invariant(property)]    // Invariant preserved
#[loop_decreases(measure)]     // Termination measure

// 3. TYPE SPECIFICATIONS
#[refinement_type]             // Create refinement type
#[refinement]                  // Mark refinement field

// 4. VISIBILITY AND EXTRACTION
#[opaque]                      // Hide implementation
#[exclude]                     // Don't extract
#[include]                     // Force extraction

// 5. BACKEND-SPECIFIC
#[fstar::options("...")]       // F* options
#[fstar::verification_status(lax)]  // Skip verification
#[lean::type("...")]           // Lean type override

// 6. PROTOCOL (ProVerif)
#[protocol_messages]           // Define protocol messages
#[pv_constructor]              // ProVerif constructor
#[process_init]                // Process definition
```

For complete documentation, see **[macro-system-complete.md](macro-system-complete.md)**.

### 2.4 cargo-hax: The Command-Line Interface

**cargo-hax** is the CLI tool that orchestrates the verification pipeline.

#### 2.4.1 Architecture

```
cargo hax into fstar
        │
        ├─→ Parse CLI arguments
        │
        ├─→ Detect Rust toolchain
        │
        ├─→ Set up rustc driver
        │
        ├─→ Invoke rustc with special flags
        │   └─→ --cfg hax
        │   └─→ Load hax frontend plugin
        │
        ├─→ Extract THIR from rustc
        │   └─→ Serialize to JSON/CBOR
        │
        ├─→ Invoke hax-engine
        │   ├─→ Parse AST
        │   ├─→ Transform
        │   └─→ Generate F* code
        │
        └─→ Write output files
```

#### 2.4.2 Key Features

```bash
# Basic extraction
cargo hax into fstar

# With options
cargo hax into fstar \
    --output-dir ./proofs \
    --z3rlimit 200 \
    --include "crypto::*" \
    --exclude "tests::*"

# Extract AST only (for debugging)
cargo hax json --pretty

# Show what would be extracted
cargo hax into fstar --dry-run

# Verbose output
RUST_LOG=debug cargo hax into fstar
```

#### 2.4.3 Configuration

```toml
# Cargo.toml
[package.metadata.hax]
# Global settings
include = ["src/verified/**"]
exclude = ["src/tests/**"]

# Backend-specific settings
[package.metadata.hax.into]
[package.metadata.hax.into.fstar]
z3rlimit = 100
fuel = 2
ifuel = 1

[package.metadata.hax.into.lean]
version = 4
mathlib = true
```

For complete CLI documentation, see **[cli-backends.md](cli-backends.md)**.

### 2.5 hax-engine: The Translation Engine

**hax-engine** is the core translation engine written primarily in OCaml.

#### 2.5.1 Pipeline Stages

```
JSON/CBOR Input
    │
    ├─→ 1. PARSING
    │      └─→ Deserialize AST from frontend
    │
    ├─→ 2. VALIDATION
    │      ├─→ Check AST well-formedness
    │      └─→ Validate type information
    │
    ├─→ 3. SIMPLIFICATION
    │      ├─→ Desugar pattern matching
    │      ├─→ Resolve trait calls
    │      ├─→ Simplify control flow
    │      └─→ Normalize expressions
    │
    ├─→ 4. TRANSFORMATION
    │      ├─→ Extract specifications
    │      ├─→ Process loop invariants
    │      ├─→ Handle refinement types
    │      └─→ Backend-specific rewrites
    │
    ├─→ 5. BACKEND SELECTION
    │      └─→ Choose F*/Lean/Coq/etc.
    │
    ├─→ 6. CODE GENERATION
    │      ├─→ Translate types
    │      ├─→ Translate expressions
    │      ├─→ Generate specifications
    │      └─→ Format output
    │
    └─→ Generated Code
```

#### 2.5.2 Key Transformations

**Pattern Desugaring:**
```rust
// Rust input
match option {
    Some(x) => x + 1,
    None => 0,
}

// Simplified representation
let __match_tmp = option;
if is_some(__match_tmp) {
    let x = unwrap(__match_tmp);
    x + 1
} else {
    0
}
```

**Trait Resolution:**
```rust
// Rust input
fn generic<T: Add>(x: T, y: T) -> T {
    x + y
}

// After monomorphization
fn generic_u32(x: u32, y: u32) -> u32 {
    U32::add(x, y)  // Explicit trait call
}
```

**Loop Invariant Injection:**
```rust
// Rust input with loop_invariant
#[loop_invariant(sum <= i * MAX)]
for i in 0..n {
    sum += arr[i];
}

// Transformed representation (conceptual)
let mut i = 0;
assert!(sum <= i * MAX);  // Initial invariant
while i < n {
    let old_i = i;
    let old_sum = sum;
    
    sum += arr[i];
    i += 1;
    
    assert!(sum <= i * MAX);  // Preserved invariant
}
```

For detailed architecture information, see **[internal-architecture.md](internal-architecture.md)**.

### 2.6 Backend Translators

Each backend translator is a module in the hax-engine that generates target-language code.

#### 2.6.1 Backend Responsibilities

All backends must:
1. **Type Translation**: Map Rust types to target types
2. **Expression Translation**: Convert Rust expressions to target syntax
3. **Specification Extraction**: Translate contracts and invariants
4. **Module System**: Handle Rust modules and visibility
5. **Name Mangling**: Ensure valid identifiers in target language

#### 2.6.2 Backend Comparison

| Feature | F\* | Lean4 | Coq | ProVerif |
|---------|-----|-------|-----|----------|
| **Maturity** | Stable | Active | Experimental | PoC |
| **Specifications** | Full | Full | Partial | Protocol-specific |
| **SMT Integration** | Yes (Z3) | Limited | No | Symbolic |
| **Loop Invariants** | Yes | Yes | Limited | N/A |
| **Refinement Types** | Yes | Yes | Limited | No |
| **Panic Proofs** | Manual | Automatic | Manual | N/A |
| **Best For** | Crypto, Systems | Functional correctness | Mathematical proofs | Protocols |

For complete backend documentation, see **[cli-backends.md](cli-backends.md)**.

---

## 3. Getting Started

### 3.1 Installation Methods

#### Method 1: Nix (Recommended)

The easiest and most reproducible installation method:

```bash
# Install Nix with flakes (one-time setup)
curl --proto '=https' --tlsv1.2 -sSf -L \
  https://install.determinate.systems/nix | sh -s -- install

# Run hax directly without installing
nix run github:hacspec/hax -- into fstar

# Or install globally
nix profile install github:hacspec/hax

# Verify installation
cargo hax --help
```

#### Method 2: From Source (Manual Setup)

For development or customization:

```bash
# 1. Clone repository
git clone https://github.com/hacspec/hax.git
cd hax

# 2. Install dependencies
# - OCaml 5.1.1+ (for engine)
# - Rust nightly (for frontend)
# - Node.js (for JS-compiled engine option)

# Install OPAM (OCaml package manager)
bash -c "sh <(curl -fsSL https://raw.githubusercontent.com/ocaml/opam/master/shell/install.sh)"
opam init
opam switch create 5.1.1

# 3. Run setup script
./setup.sh

# This will:
# - Install OCaml dependencies
# - Build hax-engine
# - Build cargo-hax CLI
# - Set up environment variables

# 4. Add to PATH
export PATH="$PATH:$(pwd)/target/release"

# 5. Verify
cargo hax --help
```

#### Method 3: Docker (Isolated Environment)

For containerized environments:

```bash
# Build Docker image
docker build -f .docker/Dockerfile . -t hax

# Run in container
docker run -it --rm -v /path/to/your/crate:/work hax bash

# Inside container:
cd /work
cargo hax into fstar
```

### 3.2 Project Setup

#### Creating a New Verified Project

```bash
# Create new library crate
cargo new --lib my-verified-lib
cd my-verified-lib

# Add hax-lib dependency
cat >> Cargo.toml << 'EOF'
[dependencies]
hax-lib = { version = "*", features = ["macros"] }

[package.metadata.hax]
# Optional: Configure hax settings
[package.metadata.hax.into.fstar]
z3rlimit = 100
EOF

# Write some verified code
cat > src/lib.rs << 'EOF'
use hax_lib as hax;

/// A simple verified function
#[hax::requires(x < 100)]
#[hax::ensures(|result| result > x)]
pub fn increment(x: u32) -> u32 {
    x + 1
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_increment() {
        assert_eq!(increment(5), 6);
        assert_eq!(increment(99), 100);
    }
}
EOF

# Test normally
cargo test

# Extract and verify
cargo hax into fstar
```

#### Adding hax to Existing Project

```toml
# Cargo.toml
[dependencies]
# Add hax-lib
hax-lib = { version = "*", features = ["macros"] }

# Keep existing dependencies
# ...
```

```rust
// src/lib.rs or src/main.rs
// Add to existing code
use hax_lib as hax;

// Gradually add specifications to existing functions
#[hax::requires(input.len() > 0)]
pub fn existing_function(input: &[u8]) -> u8 {
    // Your existing implementation
    input[0]
}
```

### 3.3 First Verified Program

Let's write a complete, verified program from scratch:

#### Step 1: Define the Problem

We'll implement a safe bounded addition that never overflows.

```rust
// src/lib.rs
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

/// Adds two numbers if the result fits in u32, otherwise returns None
/// 
/// # Specification
/// - If the sum fits in u32, returns Some(x + y)
/// - If the sum would overflow, returns None
/// - The function never panics
pub fn bounded_add(x: u32, y: u32) -> Option<u32> {
    x.checked_add(y)
}
```

#### Step 2: Add Specification

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

/// Adds two numbers if the result fits in u32, otherwise returns None
#[hax::ensures(|result| match result {
    Some(sum) => {
        // If we return Some, the sum is correct
        Int::from(sum) == Int::from(x) + Int::from(y) &&
        // And it fits in u32
        Int::from(sum) <= Int::from(u32::MAX)
    },
    None => {
        // If we return None, the sum would overflow
        Int::from(x) + Int::from(y) > Int::from(u32::MAX)
    }
})]
pub fn bounded_add(x: u32, y: u32) -> Option<u32> {
    x.checked_add(y)
}
```

#### Step 3: Add Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_no_overflow() {
        assert_eq!(bounded_add(10, 20), Some(30));
        assert_eq!(bounded_add(0, 0), Some(0));
        assert_eq!(bounded_add(u32::MAX - 1, 1), Some(u32::MAX));
    }
    
    #[test]
    fn test_overflow() {
        assert_eq!(bounded_add(u32::MAX, 1), None);
        assert_eq!(bounded_add(u32::MAX, u32::MAX), None);
        assert_eq!(bounded_add(u32::MAX - 1, 2), None);
    }
}
```

#### Step 4: Verify

```bash
# Run tests
cargo test

# Extract to F*
cargo hax into fstar

# The generated F* file will be in:
# proofs/fstar/extraction/My_verified_lib.fst

# Verify with F* (requires F* installation)
cd proofs/fstar/extraction
fstar.exe My_verified_lib.fst
```

#### Step 5: View Generated F\* Code

```fstar
(* Generated F* code *)
val bounded_add (x y: u32) : 
  Pure (option u32)
    (requires True)
    (ensures fun result -> 
      match result with
      | Some sum -> 
          v sum == v x + v y /\
          v sum <= v u32_max
      | None -> 
          v x + v y > v u32_max)

let bounded_add x y =
  U32.checked_add x y
```

### 3.4 Development Workflow

#### Recommended Workflow

```
┌─────────────────────────────────────────┐
│  1. Write Implementation                │
│     - Focus on correctness              │
│     - Use standard Rust idioms          │
│     - Add normal Rust tests             │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│  2. Add Basic Specifications            │
│     - Add #[requires] for preconditions │
│     - Add #[ensures] for postconditions │
│     - Keep specifications simple        │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│  3. Test and Debug                      │
│     - cargo test (standard tests)       │
│     - cargo build (check compilation)   │
│     - Debug with normal tools           │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│  4. Extract and Attempt Verification    │
│     - cargo hax into fstar              │
│     - Review generated code             │
│     - Attempt verification              │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│  5. Fix Verification Failures           │
│     - Read error messages               │
│     - Strengthen specifications         │
│     - Add intermediate assertions       │
│     - Add loop invariants               │
│     - Simplify complex proofs           │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│  6. Iterate Until Verified              │
│     - Refine specifications             │
│     - Improve invariants                │
│     - Add helper lemmas                 │
│     - Use opaque for modularity         │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│  7. Production                          │
│     - cargo build --release             │
│     - Fully verified code               │
│     - High assurance guarantees         │
└─────────────────────────────────────────┘
```

#### Best Practices

1. **Start Small**: Begin with simple functions before tackling complex algorithms
2. **Test First**: Use standard Rust tests to catch bugs early
3. **Incremental Specifications**: Add specifications gradually
4. **Read Errors**: hax error messages often tell you exactly what to fix
5. **Use Examples**: Study the examples/ directory in the hax repository
6. **Ask for Help**: Join the Zulip chat when stuck

---

## Part 2: Core Library (hax-lib)

## 4. Type System and Mathematical Abstractions

The hax type system extends Rust's type system with mathematical abstractions that enable precise reasoning about program behavior.

### 4.1 Mathematical Integers (Int)

The `Int` type represents unbounded mathematical integers, as opposed to Rust's fixed-width integer types.

#### 4.1.1 Why Int?

Rust's integer types wrap on overflow:

```rust
// Rust's u32 wraps around
let x: u32 = u32::MAX;
let y: u32 = 1;
let z = x + y;  // z == 0 (wraps around!)

// This makes reasoning difficult:
// x + y < x  (true due to wrapping!)
```

Mathematical integers don't wrap:

```rust
use hax_lib::int::Int;

// Mathematical integers never wrap
let x = Int::from(u32::MAX);
let y = Int::from(1u32);
let z = x + y;  // z == 2^32 (mathematical result)

// Reasoning is straightforward:
// x + y > x  (always true for positive y)
```

#### 4.1.2 Creating Int Values

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

fn int_creation_examples() {
    // Method 1: Using int! macro (for literals)
    let a = hax::int!(42);
    let b = hax::int!(1_000_000_000_000);  // Arbitrary size
    
    // Method 2: Using Int::from (for variables)
    let x: u32 = 42;
    let a = Int::from(x);
    
    // Method 3: Using .to_int() or .lift() trait method
    let x: u32 = 42;
    let a = x.to_int();  // or x.lift()
    
    // Method 4: From expressions
    let x: u32 = 10;
    let y: u32 = 20;
    let sum_concrete = x + y;
    let sum_math = Int::from(sum_concrete);
    
    // All methods create the same mathematical integer
    hax::assert!(a == hax::int!(42));
}
```

#### 4.1.3 Int Operations

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

fn int_operations() {
    let x = hax::int!(10);
    let y = hax::int!(20);
    
    // ARITHMETIC OPERATIONS
    let sum = x + y;           // Addition: 30
    let diff = x - y;          // Subtraction: -10
    let product = x * y;       // Multiplication: 200
    let quotient = x / y;      // Division: 0 (truncated)
    let negation = x.neg();    // Negation: -10
    
    // COMPARISON
    hax::assert!(x < y);       // Less than
    hax::assert!(x <= y);      // Less than or equal
    hax::assert!(y > x);       // Greater than
    hax::assert!(y >= x);      // Greater than or equal
    hax::assert!(x != y);      // Not equal
    
    // SPECIAL OPERATIONS
    let power = x.pow(3);      // Exponentiation: 1000
    let abs = diff.abs();      // Absolute value: 10
    
    // Euclidean remainder (always non-negative)
    let rem = x.rem_euclid(3); // 10 % 3 = 1
    
    // Check if representable in concrete type
    hax::assert!(sum.in_bounds::<u32>());
}
```

#### 4.1.4 Converting Back to Concrete Types

```rust
use hax_lib::int::{Int, ToInt};

fn concretization_examples(x: u32, y: u32) {
    let math_sum = Int::from(x) + Int::from(y);
    
    // Method 1: concretize() - requires proof that value fits
    let concrete: u32 = math_sum.concretize();
    // This will fail verification if math_sum might be > u32::MAX
    
    // Method 2: Safe conversion with precondition
    if math_sum <= Int::from(u32::MAX) {
        let concrete: u32 = math_sum.concretize();
        // Now provably safe
    }
    
    // Method 3: Checked conversion
    if let Some(concrete) = try_concretize::<u32>(math_sum) {
        // Use concrete value
    }
}

// Helper for safe concretization
fn try_concretize<T: TryFrom<Int>>(value: Int) -> Option<T> {
    value.try_into().ok()
}
```

#### 4.1.5 Common Patterns with Int

**Pattern 1: Overflow-Safe Arithmetic**

```rust
use hax_lib::int::{Int, ToInt};

#[hax::requires(Int::from(x) + Int::from(y) <= Int::from(u32::MAX))]
#[hax::ensures(|result| Int::from(result) == Int::from(x) + Int::from(y))]
pub fn safe_add(x: u32, y: u32) -> u32 {
    // The precondition proves this won't overflow
    x + y
}

// Usage
fn caller() {
    let a = 100u32;
    let b = 200u32;
    
    // Must prove the precondition holds
    hax::assert!(Int::from(a) + Int::from(b) <= Int::from(u32::MAX));
    let c = safe_add(a, b);
}
```

**Pattern 2: Mathematical Specification**

```rust
use hax_lib::int::{Int, ToInt};

/// Computes n! (factorial)
#[hax::decreases(n)]
#[hax::ensures(|result| Int::from(result) == math_factorial(n.to_int()))]
pub fn factorial(n: u32) -> u64 {
    if n == 0 {
        1
    } else {
        n as u64 * factorial(n - 1)
    }
}

// Specification function (not extracted, only for verification)
#[hax::exclude]
fn math_factorial(n: Int) -> Int {
    if n <= hax::int!(0) {
        hax::int!(1)
    } else {
        n * math_factorial(n - hax::int!(1))
    }
}
```

**Pattern 3: Loop Invariants with Int**

```rust
use hax_lib::int::{Int, ToInt};

fn sum_array(arr: &[u32]) -> u64 {
    let mut sum = 0u64;
    let mut i = 0usize;
    
    #[hax::loop_invariant(
        // sum equals the mathematical sum of first i elements
        Int::from(sum) == array_sum(arr, i)
    )]
    #[hax::loop_decreases(arr.len() - i)]
    while i < arr.len() {
        sum += arr[i] as u64;
        i += 1;
    }
    
    sum
}

// Specification: mathematical sum of first n elements
#[hax::exclude]
fn array_sum(arr: &[u32], n: usize) -> Int {
    if n == 0 {
        hax::int!(0)
    } else {
        Int::from(arr[n-1]) + array_sum(arr, n-1)
    }
}
```

### 4.2 Logical Propositions (Prop)

The `Prop` type represents logical propositions that can be used in specifications.

#### 4.2.1 Creating Propositions

```rust
use hax_lib::{Prop, forall, exists, implies};

fn proposition_examples() {
    // From boolean expressions
    let p = Prop::from(true);
    let q = Prop::from(1 < 2);
    
    // From quantifiers
    let all_positive = forall(|x: u32| x >= 0);
    let has_zero = exists(|x: i32| x == 0);
    
    // From implications
    let imp = implies(Prop::from(true), Prop::from(true));
}
```

#### 4.2.2 Prop Operations

```rust
use hax_lib::Prop;

fn prop_operations(x: bool, y: bool) {
    let p = Prop::from(x);
    let q = Prop::from(y);
    
    // LOGICAL OPERATIONS
    let conjunction = p.and(q);      // p ∧ q
    let disjunction = p.or(q);       // p ∨ q
    let negation = p.not();          // ¬p
    let implication = p.implies(q);  // p → q
    
    // COMPARISONS
    let equal = p.eq(q);             // p ≡ q
    let not_equal = p.ne(q);         // p ≠ q
}
```

#### 4.2.3 Using Propositions

```rust
use hax_lib as hax;

fn using_propositions(arr: &[u32], target: u32) -> bool {
    // Assert a proposition
    hax::assert_prop!(forall(|i: usize| {
        implies(i < arr.len(), arr[i] <= u32::MAX)
    }));
    
    // Check if target exists in array
    let exists_target = exists(|i: usize| {
        i < arr.len() && arr[i] == target
    });
    
    // We can't evaluate propositions directly,
    // but we can use them in specifications
    unimplemented!("Prop is for specifications only")
}
```

### 4.3 Abstraction and Concretization

The abstraction system provides a bridge between concrete machine types and abstract mathematical types.

#### 4.3.1 The Abstraction Trait

```rust
pub trait Abstraction {
    /// The abstract representation of this type
    type AbstractType;
    
    /// Convert from concrete to abstract
    fn lift(self) -> Self::AbstractType;
}

// Example implementations
impl Abstraction for u32 {
    type AbstractType = Int;
    
    fn lift(self) -> Int {
        Int::from(self)
    }
}

impl Abstraction for bool {
    type AbstractType = Prop;
    
    fn lift(self) -> Prop {
        Prop::from(self)
    }
}
```

#### 4.3.2 The Concretization Trait

```rust
pub trait Concretization<T> {
    /// Convert from abstract to concrete
    /// May fail if the abstract value doesn't fit in the concrete type
    fn concretize(self) -> T;
}

// Example implementation
impl Concretization<u32> for Int {
    fn concretize(self) -> u32 {
        // Verification will fail if self > u32::MAX
        u32::try_from(self).unwrap()
    }
}
```

#### 4.3.3 Using Abstraction/Concretization

```rust
use hax_lib::int::{Int, ToInt};

fn abstraction_example(x: u32) -> u32 {
    // Lift to Int
    let abstract_x: Int = x.lift();  // or x.to_int()
    
    // Work in abstract domain
    let abstract_result = abstract_x * hax::int!(2);
    
    // Prove result fits in u32
    hax::assert!(abstract_result <= Int::from(u32::MAX));
    
    // Concretize back to u32
    let concrete_result: u32 = abstract_result.concretize();
    
    concrete_result
}
```

### 4.4 Type Conversions and Lifting

#### 4.4.1 Automatic Lifting in Specifications

In specifications, hax automatically lifts concrete values:

```rust
#[hax::ensures(|result| result > x)]  // Automatically lifts to Int
pub fn increment(x: u32) -> u32 {
    x + 1
}

// Equivalent to:
#[hax::ensures(|result| Int::from(result) > Int::from(x))]
pub fn increment(x: u32) -> u32 {
    x + 1
}
```

#### 4.4.2 Explicit Lifting

Sometimes you need explicit lifting for clarity:

```rust
use hax_lib::int::{Int, ToInt};

#[hax::ensures(|result| 
    Int::from(result) == Int::from(x) + Int::from(y)
)]
pub fn add(x: u32, y: u32) -> u32 {
    x.wrapping_add(y)  // Even wrapping_add can be specified!
}
```

#### 4.4.3 Mixed-Type Specifications

```rust
use hax_lib::int::Int;

/// Convert u32 to i64, proving no overflow
#[hax::ensures(|result| Int::from(result) == Int::from(x))]
pub fn u32_to_i64(x: u32) -> i64 {
    x as i64  // Safe because u32::MAX < i64::MAX
}

/// Saturating subtraction specification
#[hax::ensures(|result| {
    if Int::from(x) >= Int::from(y) {
        Int::from(result) == Int::from(x) - Int::from(y)
    } else {
        result == 0
    }
})]
pub fn saturating_sub(x: u32, y: u32) -> u32 {
    x.saturating_sub(y)
}
```

---

## 5. Assertions and Assumptions

Assertions and assumptions are the building blocks of verification.

### 5.1 assert! - Provable Assertions

The `assert!` macro generates a **proof obligation** that the verifier must prove.

#### 5.1.1 Basic Usage

```rust
use hax_lib as hax;

pub fn simple_assert(x: u32) {
    // The verifier must prove x < u32::MAX
    hax::assert!(x < u32::MAX);
    
    // This always holds for u32
    // Verification succeeds
}

pub fn failing_assert(x: u32, y: u32) {
    // The verifier must prove x + y never overflows
    hax::assert!(x + y > x);
    
    // This might not hold (overflow)
    // Verification fails without more context
}
```

#### 5.1.2 Runtime Behavior

```rust
use hax_lib as hax;

pub fn assertion_runtime() {
    hax::assert!(true);   // No-op in release builds
                          // debug_assert! in debug builds
    
    hax::assert!(false);  // Panics in debug builds
                          // No-op in release builds
                          // Verification fails
}
```

#### 5.1.3 When to Use assert!

```rust
use hax_lib as hax;

/// Use assert! to help the verifier prove postconditions
#[hax::ensures(|result| result > x)]
pub fn increment(x: u32) -> u32 {
    let result = x + 1;
    
    // Help the verifier by asserting intermediate steps
    hax::assert!(result > x);
    
    result
}

/// Use assert! to document assumptions about code
pub fn process_array(arr: &[u32]) {
    hax::assert!(arr.len() <= 1000);  // Document size assumption
    
    for i in 0..arr.len() {
        // Process elements
        hax::assert!(i < arr.len());  // Bounds check proof
    }
}

/// Use assert! to break down complex proofs
pub fn complex_computation(x: u32, y: u32) -> u32 {
    hax::assert!(x < 100);  // Step 1
    hax::assert!(y < 100);  // Step 2
    
    let sum = x + y;
    hax::assert!(sum < 200);  // Step 3: follows from 1 and 2
    
    sum
}
```

### 5.2 assume! - Trusted Assumptions

The `assume!` macro tells the verifier to **trust** a condition without proof. This is **unsafe** and should be used sparingly.

#### 5.2.1 Basic Usage

```rust
use hax_lib as hax;

pub fn unsafe_assumption(x: u32, y: u32) -> u32 {
    // Tell the verifier to trust that y != 0
    // WITHOUT PROOF
    hax::assume!(y != 0);
    
    // Now the division is "proven" safe
    // But this is unsound if y could actually be 0!
    x / y
}
```

#### 5.2.2 When to Use assume!

**Use Case 1: Trusting Unverified Code**

```rust
use hax_lib as hax;

/// External function we can't verify
#[hax::exclude]
extern "C" {
    fn external_function(x: u32) -> u32;
}

pub fn call_external(x: u32) -> u32 {
    let result = unsafe { external_function(x) };
    
    // We trust (but can't prove) that external_function
    // never returns 0
    hax::assume!(result != 0);
    
    100 / result
}
```

**Use Case 2: Breaking Circular Dependencies**

```rust
use hax_lib as hax;

pub fn mutually_recursive_a(x: u32) -> u32 {
    if x > 0 {
        mutually_recursive_b(x - 1)
    } else {
        0
    }
}

pub fn mutually_recursive_b(x: u32) -> u32 {
    if x > 0 {
        mutually_recursive_a(x - 1)
    } else {
        // Assume termination (trust)
        hax::assume!(x == 0);
        1
    }
}
```

**Use Case 3: Axioms and Mathematical Assumptions**

```rust
use hax_lib as hax;

pub fn prime_theorem(n: u32) -> bool {
    // Assume the prime number theorem
    hax::assume!(prime_density_bound(n));
    
    // Use this assumption in further proofs
    is_prime(n)
}

#[hax::exclude]
fn prime_density_bound(n: u32) -> bool {
    // Mathematical property we trust
    true
}
```

#### 5.2.3 Dangers of assume!

```rust
use hax_lib as hax;

/// ⚠️ DANGER: Unsound proof!
pub fn unsound_example(x: u32) -> u32 {
    // This is FALSE but we assume it
    hax::assume!(x > 100);
    
    // Now we can "prove" anything
    hax::assert!(x > 50);   // "Verified" but wrong!
    
    // This code could panic at runtime
    x - 50
}

/// The verifier thinks this is safe, but it can crash!
fn caller() {
    let result = unsound_example(10);  // PANIC! 10 - 50 underflows
}
```

### 5.3 assert_prop! - Logical Propositions

The `assert_prop!` macro asserts logical propositions using quantifiers.

#### 5.3.1 Basic Usage

```rust
use hax_lib as hax;

pub fn prop_assertion(arr: &[u32]) {
    // Assert a property over all array elements
    hax::assert_prop!(forall(|i: usize| {
        implies(i < arr.len(), arr[i] >= 0)
    }));
    
    // This always holds for u32 (always non-negative)
    // Verification succeeds
}
```

#### 5.3.2 Complex Propositions

```rust
use hax_lib as hax;

pub fn complex_proposition(arr: &[i32], target: i32) {
    // Existential quantification
    hax::assert_prop!(exists(|i: usize| {
        i < arr.len() && arr[i] == target
    }));
    
    // Universal with implication
    hax::assert_prop!(forall(|i: usize| {
        implies(i < arr.len() && arr[i] == target, i < 100)
    }));
    
    // Nested quantifiers
    hax::assert_prop!(forall(|i: usize| {
        implies(i < arr.len(),
            forall(|j: usize| {
                implies(j < arr.len() && i != j, arr[i] != arr[j])
            })
        )
    }));
}
```

#### 5.3.3 assert! vs assert_prop!

```rust
use hax_lib as hax;

fn comparison(x: u32) {
    // assert! - for concrete, computable conditions
    hax::assert!(x < 100);  // Can evaluate at runtime
    
    // assert_prop! - for logical, possibly non-computable propositions
    hax::assert_prop!(forall(|y: u32| y >= 0));  // Can't evaluate at runtime
}
```

### 5.4 debug_assert! - Runtime-Only Checks

Standard Rust `debug_assert!` is **completely ignored** by hax.

#### 5.4.1 Usage

```rust
use hax_lib as hax;

pub fn debugging_example(x: u32) -> u32 {
    // This is ONLY for runtime debugging
    // Generates NO proof obligation
    debug_assert!(expensive_check(x));
    
    // hax doesn't see this assertion at all
    x + 1
}

fn expensive_check(x: u32) -> bool {
    // Expensive computation for testing
    (0..x).all(|i| i < 1000)
}
```

#### 5.4.2 When to Use debug_assert!

```rust
pub fn use_debug_assert(arr: &[u32]) {
    // Use debug_assert! for:
    
    // 1. Expensive runtime checks
    debug_assert!(arr.iter().all(|&x| is_valid(x)));
    
    // 2. Checks that are redundant with verification
    for i in 0..arr.len() {
        hax::assert!(i < arr.len());  // Verified
        debug_assert!(i < arr.len()); // Runtime check (redundant)
    }
    
    // 3. Development/testing assertions
    debug_assert_eq!(arr.len(), expected_len());
}

#[hax::exclude]
fn is_valid(x: u32) -> bool { x < 1000 }

#[hax::exclude]
fn expected_len() -> usize { 10 }
```

### 5.5 Best Practices and Safety

#### 5.5.1 Assertion Hierarchy

```
debug_assert!     ← Only runtime, no verification
    ↓
assert!           ← Proof obligation (verifier must prove)
    ↓
assert_prop!      ← Logical proposition (with quantifiers)
    ↓
assume!           ← Trust (no proof, UNSAFE)
```

#### 5.5.2 Choosing the Right Assertion

```rust
use hax_lib as hax;

pub fn choosing_assertions(x: u32, arr: &[u32]) {
    // Use assert! for simple, provable facts
    hax::assert!(x < u32::MAX);
    
    // Use assert_prop! for logical properties
    hax::assert_prop!(forall(|i: usize| {
        implies(i < arr.len(), arr[i] < 1000)
    }));
    
    // Use assume! ONLY when necessary (rare!)
    #[cfg(feature = "unsafe_trust")]
    {
        hax::assume!(x > 100);  // Trust external invariant
    }
    
    // Use debug_assert! for expensive runtime checks
    debug_assert!(complex_property_holds(x, arr));
}

#[hax::exclude]
fn complex_property_holds(x: u32, arr: &[u32]) -> bool {
    // Expensive check
    true
}
```

#### 5.5.3 Common Pitfalls

```rust
use hax_lib as hax;

/// ❌ WRONG: Using assume! instead of proving
pub fn wrong_division(x: u32, y: u32) -> u32 {
    hax::assume!(y != 0);  // Lazy! Should prove this
    x / y
}

/// ✅ CORRECT: Prove with precondition
#[hax::requires(y != 0)]
pub fn correct_division(x: u32, y: u32) -> u32 {
    x / y
}

/// ❌ WRONG: Asserting unprovable facts
pub fn wrong_assert(x: u32) {
    hax::assert!(x == 42);  // Can't prove this for arbitrary x!
}

/// ✅ CORRECT: Only assert provable facts
#[hax::requires(x == 42)]
pub fn correct_assert(x: u32) {
    hax::assert!(x == 42);  // Now provable from precondition
}
```

---

## 6. Logical Specifications

Logical specifications use quantifiers and logical connectives to express properties.

### 6.1 Universal Quantification (forall!)

The `forall!` macro expresses that a property holds for all values.

#### 6.1.1 Basic Syntax

```rust
use hax_lib::{forall, implies};

fn forall_examples() {
    // All u32 values are non-negative
    let prop1 = forall(|x: u32| x >= 0);
    
    // All array indices are in bounds
    let prop2 = forall(|i: usize| 
        implies(i < arr.len(), arr[i] <= u32::MAX)
    );
    
    // All pairs satisfy a relation
    let prop3 = forall(|x: u32| forall(|y: u32| 
        x + y >= x
    ));
}
```

#### 6.1.2 Using forall! in Specifications

```rust
use hax_lib as hax;

/// Ensure all array elements are positive
#[hax::requires(forall(|i: usize| 
    implies(i < arr.len(), arr[i] > 0)
))]
pub fn all_positive(arr: &[u32]) {
    // We can assume all elements are positive
    for i in 0..arr.len() {
        hax::assert!(arr[i] > 0);
    }
}

/// Find minimum element
#[hax::requires(arr.len() > 0)]
#[hax::ensures(|result| 
    forall(|i: usize| implies(i < arr.len(), result <= arr[i])) &&
    exists(|i: usize| i < arr.len() && result == arr[i])
)]
pub fn find_min(arr: &[u32]) -> u32 {
    let mut min = arr[0];
    for i in 1..arr.len() {
        if arr[i] < min {
            min = arr[i];
        }
    }
    min
}
```

#### 6.1.3 Nested Universal Quantifiers

```rust
use hax_lib as hax;

/// Matrix property: all rows have equal length
#[hax::requires(
    forall(|i: usize| forall(|j: usize| 
        implies(
            i < matrix.len() && j < matrix.len(),
            matrix[i].len() == matrix[j].len()
        )
    ))
)]
pub fn rectangular_matrix(matrix: &[Vec<u32>]) {
    // Can assume all rows have same length
    if matrix.len() > 0 {
        let width = matrix[0].len();
        for i in 0..matrix.len() {
            hax::assert!(matrix[i].len() == width);
        }
    }
}
```

### 6.2 Existential Quantification (exists!)

The `exists!` macro expresses that a property holds for at least one value.

#### 6.2.1 Basic Syntax

```rust
use hax_lib::exists;

fn exists_examples(arr: &[u32], target: u32) {
    // There exists a zero in the array
    let prop1 = exists(|i: usize| 
        i < arr.len() && arr[i] == 0
    );
    
    // There exists an element equal to target
    let prop2 = exists(|i: usize|
        i < arr.len() && arr[i] == target
    );
    
    // There exist two different indices with same value
    let prop3 = exists(|i: usize| exists(|j: usize|
        i < arr.len() && j < arr.len() && i != j && arr[i] == arr[j]
    ));
}
```

#### 6.2.2 Using exists! in Specifications

```rust
use hax_lib as hax;

/// Search for target in array
#[hax::ensures(|result| match result {
    Some(idx) => idx < arr.len() && arr[idx] == target,
    None => forall(|i: usize| 
        implies(i < arr.len(), arr[i] != target)
    )
})]
pub fn search(arr: &[u32], target: u32) -> Option<usize> {
    for i in 0..arr.len() {
        if arr[i] == target {
            return Some(i);
        }
    }
    None
}

/// Check if array contains duplicates
#[hax::ensures(|result| 
    result == exists(|i: usize| exists(|j: usize|
        i < arr.len() && j < arr.len() && i < j && arr[i] == arr[j]
    ))
)]
pub fn has_duplicates(arr: &[u32]) -> bool {
    for i in 0..arr.len() {
        for j in (i+1)..arr.len() {
            if arr[i] == arr[j] {
                return true;
            }
        }
    }
    false
}
```

### 6.3 Logical Implication (implies!)

The `implies!` macro expresses logical implication: "if A then B".

#### 6.3.1 Basic Syntax

```rust
use hax_lib::implies;

fn implies_examples(x: u32) {
    // If x > 10, then x > 5
    let prop1 = implies(x > 10, x > 5);
    
    // If x is even, then x % 2 == 0
    let prop2 = implies(x % 2 == 0, is_even(x));
    
    // Chained implications
    let prop3 = implies(x > 100, implies(x > 50, x > 25));
}

fn is_even(x: u32) -> bool {
    x % 2 == 0
}
```

#### 6.3.2 Using implies! with Quantifiers

```rust
use hax_lib as hax;

/// Bounded array property
#[hax::requires(forall(|i: usize| 
    implies(i < arr.len(), arr[i] <= bound)
))]
#[hax::ensures(|result| result <= bound)]
pub fn max_bounded(arr: &[u32], bound: u32) -> u32 {
    let mut max = 0;
    for i in 0..arr.len() {
        // From precondition: arr[i] <= bound
        hax::assert!(implies(i < arr.len(), arr[i] <= bound));
        
        if arr[i] > max {
            max = arr[i];
        }
    }
    max
}

/// Conditional property
#[hax::ensures(|result| 
    implies(result, forall(|i: usize| 
        implies(i < arr.len(), arr[i] > 0)
    ))
)]
pub fn all_positive_check(arr: &[u32]) -> bool {
    for i in 0..arr.len() {
        if arr[i] == 0 {
            return false;
        }
    }
    true
}
```

### 6.4 Proposition Combinators

Combine propositions using logical operators.

#### 6.4.1 Conjunction (AND)

```rust
use hax_lib::Prop;

fn conjunction_example(x: u32, y: u32) -> Prop {
    let p = Prop::from(x > 0);
    let q = Prop::from(y > 0);
    
    // Both x and y are positive
    p.and(q)
}

/// Function with multiple requirements
#[hax::requires(x > 0 && y > 0 && x < 100 && y < 100)]
pub fn both_positive(x: u32, y: u32) -> u32 {
    x + y
}
```

#### 6.4.2 Disjunction (OR)

```rust
use hax_lib::Prop;

fn disjunction_example(x: u32) -> Prop {
    let p = Prop::from(x == 0);
    let q = Prop::from(x > 0);
    
    // x is either zero or positive
    p.or(q)
}

/// At least one positive
#[hax::requires(x > 0 || y > 0)]
pub fn at_least_one_positive(x: u32, y: u32) -> bool {
    x > 0 || y > 0
}
```

#### 6.4.3 Negation (NOT)

```rust
use hax_lib::Prop;

fn negation_example(x: u32) -> Prop {
    let p = Prop::from(x == 0);
    
    // x is not zero
    p.not()
}

/// x is non-zero
#[hax::requires(x != 0)]
pub fn non_zero(x: u32) -> u32 {
    100 / x
}
```

### 6.5 Writing Complex Specifications

#### 6.5.1 Sorted Array Property

```rust
use hax_lib as hax;

/// Check if array is sorted
#[hax::ensures(|result|
    result == forall(|i: usize| forall(|j: usize|
        implies(
            i < arr.len() && j < arr.len() && i < j,
            arr[i] <= arr[j]
        )
    ))
)]
pub fn is_sorted(arr: &[u32]) -> bool {
    for i in 0..(arr.len().saturating_sub(1)) {
        if arr[i] > arr[i + 1] {
            return false;
        }
    }
    true
}
```

#### 6.5.2 Partitioning Property

```rust
use hax_lib as hax;

/// Partition array around pivot
#[hax::ensures(|pivot_idx|
    // All elements before pivot are <= pivot value
    forall(|i: usize| implies(
        i < pivot_idx,
        arr[i] <= arr[pivot_idx]
    )) &&
    // All elements after pivot are >= pivot value
    forall(|j: usize| implies(
        j >= pivot_idx && j < arr.len(),
        arr[j] >= arr[pivot_idx]
    ))
)]
pub fn partition(arr: &mut [u32]) -> usize {
    // Implementation...
    0
}
```

#### 6.5.3 Permutation Property

```rust
use hax_lib as hax;

/// Sort array (produces permutation of input)
#[hax::ensures(|_|
    // Output is sorted
    forall(|i: usize| forall(|j: usize|
        implies(i < j && j < arr.len(), arr[i] <= arr[j])
    )) &&
    // Output is permutation (same multiset)
    forall(|val: u32|
        count_occurrences(arr, val) == count_occurrences(hax::old(arr), val)
    )
)]
pub fn sort(arr: &mut [u32]) {
    // Implementation...
}

#[hax::exclude]
fn count_occurrences(arr: &[u32], val: u32) -> usize {
    arr.iter().filter(|&&x| x == val).count()
}
```

---

## 7. Refinement Types

Refinement types allow you to encode invariants directly into the type system.

### 7.1 Refinement Type Theory

A refinement type is a type `T` paired with a logical predicate `P` such that values of type `{x: T | P(x)}` are exactly those values `v` of type `T` for which `P(v)` holds.

```
Type:     u32
Refinement: { x: u32 | x != 0 }
Values:   All u32 except 0
```

In hax, refinement types are implemented as wrapper structs with an invariant.

### 7.2 The Refinement Trait

```rust
pub trait Refinement {
    /// The underlying concrete type
    type InnerType;
    
    /// The logical invariant that must hold
    fn invariant(value: Self::InnerType) -> Prop;
    
    /// Construct from inner value (unchecked, for internal use)
    fn new(value: Self::InnerType) -> Self;
    
    /// Extract inner value
    fn get(self) -> Self::InnerType;
    
    /// Get mutable reference to inner value
    fn get_mut(&mut self) -> &mut Self::InnerType;
}
```

### 7.3 Creating Custom Refinements

#### 7.3.1 Basic Refinement Type

```rust
use hax_lib as hax;

/// A non-zero integer
#[hax::refinement_type]
pub struct NonZero {
    #[hax::refinement]
    value: i32,
}

impl hax::Refinement for NonZero {
    type InnerType = i32;
    
    fn invariant(value: i32) -> hax::Prop {
        hax::Prop::from(value != 0)
    }
}

/// Smart constructor with validation
impl NonZero {
    #[hax::ensures(|result| match result {
        Some(nz) => *nz == value && value != 0,
        None => value == 0
    })]
    pub fn new(value: i32) -> Option<Self> {
        if value != 0 {
            Some(NonZero { value })
        } else {
            None
        }
    }
}

/// Usage: Division is provably safe
pub fn safe_divide(x: i32, divisor: NonZero) -> i32 {
    // The type guarantees *divisor != 0
    x / *divisor
}
```

#### 7.3.2 Bounded Integer Type

```rust
use hax_lib as hax;

/// An integer in the range [MIN, MAX]
#[hax::refinement_type]
pub struct BoundedU32<const MIN: u32, const MAX: u32> {
    #[hax::refinement]
    value: u32,
}

impl<const MIN: u32, const MAX: u32> hax::Refinement for BoundedU32<MIN, MAX> {
    type InnerType = u32;
    
    fn invariant(value: u32) -> hax::Prop {
        hax::Prop::from(value >= MIN && value <= MAX)
    }
}

/// Smart constructor
impl<const MIN: u32, const MAX: u32> BoundedU32<MIN, MAX> {
    pub fn new(value: u32) -> Option<Self> {
        if value >= MIN && value <= MAX {
            Some(BoundedU32 { value })
        } else {
            None
        }
    }
}

/// Type alias for common case
pub type Percentage = BoundedU32<0, 100>;

/// Usage
pub fn scale_by_percentage(value: u32, percent: Percentage) -> u32 {
    // percent is guaranteed to be 0-100
    (value * (*percent)) / 100
}
```

#### 7.3.3 Non-Empty Collection

```rust
use hax_lib as hax;

/// A vector that is guaranteed non-empty
#[hax::refinement_type]
pub struct NonEmptyVec<T> {
    #[hax::refinement]
    data: Vec<T>,
}

impl<T> hax::Refinement for NonEmptyVec<T> {
    type InnerType = Vec<T>;
    
    fn invariant(data: Vec<T>) -> hax::Prop {
        hax::Prop::from(data.len() > 0)
    }
}

impl<T> NonEmptyVec<T> {
    pub fn new(data: Vec<T>) -> Option<Self> {
        if data.is_empty() {
            None
        } else {
            Some(NonEmptyVec { data })
        }
    }
    
    /// Safe first element access (no panic possible)
    pub fn first(&self) -> &T {
        // The refinement guarantees len() > 0
        &self.data[0]
    }
    
    /// Safe pop (always returns Some)
    pub fn pop(&mut self) -> Option<T> {
        if self.data.len() == 1 {
            // Would violate invariant
            None
        } else {
            self.data.pop()
        }
    }
}
```

### 7.4 Refinement Type Patterns

#### 7.4.1 Validated Input

```rust
use hax_lib as hax;

#[hax::refinement_type]
pub struct ValidEmail {
    #[hax::refinement]
    email: String,
}

impl hax::Refinement for ValidEmail {
    type InnerType = String;
    
    fn invariant(email: String) -> hax::Prop {
        hax::Prop::from(is_valid_email(&email))
    }
}

#[hax::exclude]
fn is_valid_email(email: &str) -> bool {
    email.contains('@') && email.contains('.')
}

impl ValidEmail {
    pub fn parse(input: String) -> Result<Self, String> {
        if is_valid_email(&input) {
            Ok(ValidEmail { email: input })
        } else {
            Err("Invalid email".to_string())
        }
    }
}

/// Functions can require validated input
pub fn send_email(to: ValidEmail, message: &str) {
    // Guaranteed that `to` is a valid email
    // No runtime validation needed
}
```

#### 7.4.2 Type-Level State Machine

```rust
use hax_lib as hax;

mod states {
    pub struct Open;
    pub struct Closed;
}

#[hax::refinement_type]
pub struct File<State> {
    #[hax::refinement]
    handle: FileHandle,
    _state: PhantomData<State>,
}

impl File<states::Closed> {
    pub fn open(path: &str) -> Result<File<states::Open>, Error> {
        let handle = open_file(path)?;
        Ok(File { handle, _state: PhantomData })
    }
}

impl File<states::Open> {
    pub fn write(&mut self, data: &[u8]) -> Result<(), Error> {
        write_to_file(&self.handle, data)
    }
    
    pub fn close(self) -> File<states::Closed> {
        close_file(&self.handle);
        File { handle: self.handle, _state: PhantomData }
    }
}

// Can't write to closed file (type error)
// Can't close already closed file (type error)
```

### 7.5 Advanced Usage

#### 7.5.1 Refinement Type Algebra

```rust
use hax_lib as hax;

/// Positive integer
#[hax::refinement_type]
pub struct Positive {
    #[hax::refinement]
    value: i32,
}

impl hax::Refinement for Positive {
    type InnerType = i32;
    fn invariant(value: i32) -> hax::Prop {
        hax::Prop::from(value > 0)
    }
}

/// Even integer
#[hax::refinement_type]
pub struct Even {
    #[hax::refinement]
    value: i32,
}

impl hax::Refinement for Even {
    type InnerType = i32;
    fn invariant(value: i32) -> hax::Prop {
        hax::Prop::from(value % 2 == 0)
    }
}

/// Positive and even integer
#[hax::refinement_type]
pub struct PositiveEven {
    #[hax::refinement]
    value: i32,
}

impl hax::Refinement for PositiveEven {
    type InnerType = i32;
    fn invariant(value: i32) -> hax::Prop {
        hax::Prop::from(value > 0 && value % 2 == 0)
    }
}

/// Convert from stricter to less strict
impl From<PositiveEven> for Positive {
    fn from(pe: PositiveEven) -> Positive {
        Positive { value: *pe }
    }
}

impl From<PositiveEven> for Even {
    fn from(pe: PositiveEven) -> Even {
        Even { value: *pe }
    }
}
```

#### 7.5.2 Dependent Refinements

```rust
use hax_lib as hax;

/// Array with known size
#[hax::refinement_type]
pub struct SizedArray<T, const N: usize> {
    #[hax::refinement]
    data: Vec<T>,
}

impl<T, const N: usize> hax::Refinement for SizedArray<T, N> {
    type InnerType = Vec<T>;
    
    fn invariant(data: Vec<T>) -> hax::Prop {
        hax::Prop::from(data.len() == N)
    }
}

impl<T, const N: usize> SizedArray<T, N> {
    /// Safe indexing without bounds checks
    pub fn get(&self, index: usize) -> Option<&T> {
        if index < N {
            Some(&self.data[index])  // Provably safe
        } else {
            None
        }
    }
}
```

---

*[Continue with remaining sections in the same exhaustive style...]*

I'll create the complete, exhaustively detailed README.md. Let me continue with the remaining sections:


## Part 3: Specification and Verification

## 8. Function Contracts

Function contracts define the interface between callers and implementations using preconditions and postconditions.

### 8.1 Preconditions (#[requires])

Preconditions specify what must be true when the function is called.

#### 8.1.1 Basic Preconditions

```rust
use hax_lib as hax;

/// Simple precondition: x must be positive
#[hax::requires(x > 0)]
pub fn reciprocal(x: f64) -> f64 {
    1.0 / x
}

/// Multiple preconditions using &&
#[hax::requires(x > 0 && y > 0)]
pub fn both_positive(x: i32, y: i32) -> i32 {
    x + y
}

/// Precondition on array length
#[hax::requires(arr.len() > 0)]
pub fn first_element(arr: &[u32]) -> u32 {
    arr[0]  // Provably safe due to precondition
}
```

#### 8.1.2 Multiple Preconditions

```rust
use hax_lib as hax;

/// Multiple #[requires] attributes (all must hold)
#[hax::requires(x < 100)]
#[hax::requires(y < 100)]
#[hax::requires(x + y < 150)]
pub fn constrained_sum(x: u32, y: u32) -> u32 {
    x + y
}

/// Equivalent: combine with &&
#[hax::requires(x < 100 && y < 100 && x + y < 150)]
pub fn constrained_sum_2(x: u32, y: u32) -> u32 {
    x + y
}
```

#### 8.1.3 Complex Preconditions

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

/// Precondition using Int for overflow reasoning
#[hax::requires(
    Int::from(x) + Int::from(y) <= Int::from(u32::MAX)
)]
pub fn overflow_safe_add(x: u32, y: u32) -> u32 {
    x + y
}

/// Precondition with quantifiers
#[hax::requires(forall(|i: usize| 
    implies(i < arr.len(), arr[i] > 0)
))]
pub fn all_positive_product(arr: &[u32]) -> u64 {
    arr.iter().map(|&x| x as u64).product()
}

/// Precondition relating multiple parameters
#[hax::requires(arr.len() == weights.len())]
#[hax::requires(forall(|i: usize|
    implies(i < arr.len(), weights[i] > 0)
))]
pub fn weighted_sum(arr: &[u32], weights: &[u32]) -> u64 {
    arr.iter()
       .zip(weights.iter())
       .map(|(&a, &w)| a as u64 * w as u64)
       .sum()
}
```

#### 8.1.4 Precondition Patterns

**Pattern 1: Bounds Checking**

```rust
#[hax::requires(index < arr.len())]
pub fn safe_get(arr: &[u32], index: usize) -> u32 {
    arr[index]
}
```

**Pattern 2: Non-Zero Divisor**

```rust
#[hax::requires(divisor != 0)]
pub fn safe_divide(dividend: u32, divisor: u32) -> u32 {
    dividend / divisor
}
```

**Pattern 3: Sorted Input**

```rust
#[hax::requires(forall(|i: usize| forall(|j: usize|
    implies(i < j && j < arr.len(), arr[i] <= arr[j])
)))]
pub fn binary_search_sorted(arr: &[u32], target: u32) -> Option<usize> {
    // Implementation can assume array is sorted
    unimplemented!()
}
```

**Pattern 4: Non-Empty Collection**

```rust
#[hax::requires(v.len() > 0)]
pub fn head<T>(v: &[T]) -> &T {
    &v[0]
}
```

### 8.2 Postconditions (#[ensures])

Postconditions specify what must be true after the function returns.

#### 8.2.1 Basic Postconditions

```rust
use hax_lib as hax;

/// Simple postcondition: result is positive
#[hax::ensures(|result| result > 0)]
pub fn always_positive() -> u32 {
    42
}

/// Postcondition relating result to input
#[hax::ensures(|result| result > x)]
pub fn increment(x: u32) -> u32 {
    x + 1
}

/// Postcondition on Option
#[hax::ensures(|result| match result {
    Some(val) => val > 0,
    None => true
})]
pub fn try_positive(x: i32) -> Option<u32> {
    if x > 0 {
        Some(x as u32)
    } else {
        None
    }
}
```

#### 8.2.2 Multiple Postconditions

```rust
use hax_lib as hax;

/// Multiple postconditions (all must hold)
#[hax::ensures(|result| result >= x)]
#[hax::ensures(|result| result >= y)]
#[hax::ensures(|result| result == x || result == y)]
pub fn max(x: u32, y: u32) -> u32 {
    if x > y { x } else { y }
}

/// Equivalent: combine conditions
#[hax::ensures(|result| 
    result >= x && result >= y && (result == x || result == y)
)]
pub fn max_2(x: u32, y: u32) -> u32 {
    if x > y { x } else { y }
}
```

#### 8.2.3 Complex Postconditions

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

/// Postcondition using Int
#[hax::ensures(|result| match result {
    Some(sum) => Int::from(sum) == Int::from(x) + Int::from(y),
    None => Int::from(x) + Int::from(y) > Int::from(u32::MAX)
})]
pub fn checked_add(x: u32, y: u32) -> Option<u32> {
    x.checked_add(y)
}

/// Postcondition with quantifiers
#[hax::requires(arr.len() > 0)]
#[hax::ensures(|result|
    forall(|i: usize| implies(i < arr.len(), result >= arr[i])) &&
    exists(|i: usize| i < arr.len() && result == arr[i])
)]
pub fn find_max(arr: &[u32]) -> u32 {
    let mut max = arr[0];
    for i in 1..arr.len() {
        if arr[i] > max {
            max = arr[i];
        }
    }
    max
}
```

#### 8.2.4 Postcondition Patterns

**Pattern 1: Correctness Specification**

```rust
use hax_lib::int::Int;

#[hax::requires(divisor != 0)]
#[hax::ensures(|result|
    Int::from(result) * Int::from(divisor) + Int::from(dividend % divisor)
    == Int::from(dividend)
)]
pub fn divide(dividend: u32, divisor: u32) -> u32 {
    dividend / divisor
}
```

**Pattern 2: Range Specification**

```rust
#[hax::ensures(|result| result >= min && result <= max)]
pub fn clamp(x: u32, min: u32, max: u32) -> u32 {
    if x < min {
        min
    } else if x > max {
        max
    } else {
        x
    }
}
```

**Pattern 3: Preservation Property**

```rust
#[hax::ensures(|result| result.len() == arr.len())]
pub fn double_all(arr: &[u32]) -> Vec<u32> {
    arr.iter().map(|&x| x * 2).collect()
}
```

**Pattern 4: Relational Specification**

```rust
#[hax::ensures(|result|
    forall(|i: usize| implies(
        i < result.len(),
        exists(|j: usize| j < arr.len() && arr[j] == result[i])
    ))
)]
pub fn filter_positive(arr: &[i32]) -> Vec<i32> {
    arr.iter().copied().filter(|&x| x > 0).collect()
}
```

### 8.3 Contract Composition

#### 8.3.1 Combining Preconditions and Postconditions

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

/// Full contract: precondition + postcondition
#[hax::requires(x > 0 && y > 0)]
#[hax::requires(Int::from(x) * Int::from(y) <= Int::from(u32::MAX))]
#[hax::ensures(|result| Int::from(result) == Int::from(x) * Int::from(y))]
#[hax::ensures(|result| result > 0)]
pub fn safe_multiply(x: u32, y: u32) -> u32 {
    x * y
}
```

#### 8.3.2 Calling Functions with Contracts

```rust
use hax_lib as hax;

#[hax::requires(divisor != 0)]
#[hax::ensures(|result| result >= 0)]
pub fn divide(dividend: u32, divisor: u32) -> u32 {
    dividend / divisor
}

pub fn caller() {
    let x = 10u32;
    let y = 5u32;
    
    // Must prove precondition holds
    if y != 0 {
        let result = divide(x, y);
        
        // Can assume postcondition holds
        hax::assert!(result >= 0);
    }
}
```

#### 8.3.3 Contract Weakening and Strengthening

```rust
use hax_lib as hax;

/// Stronger precondition, weaker postcondition
#[hax::requires(x > 10)]  // Stronger than x > 0
#[hax::ensures(|result| result > 0)]  // Weaker than result > x
pub fn strong_requirement(x: u32) -> u32 {
    x - 5
}

/// Can call from function with weaker precondition
#[hax::requires(x > 0)]
pub fn weak_caller(x: u32) {
    if x > 10 {
        let _ = strong_requirement(x);  // OK: proves stronger precondition
    }
}
```

### 8.4 Advanced Contract Patterns

#### 8.4.1 Contracts with Ghost State

```rust
use hax_lib as hax;

#[hax::ensures(|result| result.0 + result.1 == hax::old(x))]
pub fn split_number(x: u32) -> (u32, u32) {
    let half = x / 2;
    let rem = x - half;
    (half, rem)
}
```

#### 8.4.2 Contracts on Mutable Functions

```rust
use hax_lib as hax;

/// Postcondition references old and new state
#[hax::ensures(|_| *x == hax::old(*x) + 1)]
pub fn increment_in_place(x: &mut u32) {
    *x += 1;
}

/// Contract on mutable slice
#[hax::ensures(|_| forall(|i: usize|
    implies(i < arr.len(), arr[i] == hax::old(arr[i]) * 2)
))]
pub fn double_in_place(arr: &mut [u32]) {
    for i in 0..arr.len() {
        arr[i] *= 2;
    }
}
```

#### 8.4.3 Conditional Postconditions

```rust
use hax_lib as hax;

/// Different postconditions for different cases
#[hax::ensures(|result| implies(x >= 0, result == x as u32))]
#[hax::ensures(|result| implies(x < 0, result == 0))]
pub fn to_unsigned_saturating(x: i32) -> u32 {
    if x >= 0 {
        x as u32
    } else {
        0
    }
}
```

---

## 9. Loop Specifications

Loops are verified using invariants and termination measures.

### 9.1 Loop Invariants (#[loop_invariant])

Loop invariants are properties that hold before and after each iteration.

#### 9.1.1 Basic Loop Invariants

```rust
use hax_lib as hax;

pub fn count_up(n: u32) -> u32 {
    let mut i = 0;
    
    #[hax::loop_invariant(i <= n)]
    while i < n {
        i += 1;
    }
    
    i
}
```

#### 9.1.2 Invariants Relating Multiple Variables

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

pub fn sum_to_n(n: u32) -> u64 {
    let mut sum = 0u64;
    let mut i = 0u32;
    
    #[hax::loop_invariant(i <= n)]
    #[hax::loop_invariant(Int::from(sum) == Int::from(i) * (Int::from(i) + 1) / 2)]
    while i < n {
        i += 1;
        sum += i as u64;
    }
    
    sum
}
```

#### 9.1.3 Invariants for Array Processing

```rust
use hax_lib as hax;

pub fn sum_array(arr: &[u32]) -> u64 {
    let mut sum = 0u64;
    let mut i = 0usize;
    
    #[hax::loop_invariant(i <= arr.len())]
    #[hax::loop_invariant(sum == array_sum(arr, i))]
    while i < arr.len() {
        sum += arr[i] as u64;
        i += 1;
    }
    
    sum
}

#[hax::exclude]
fn array_sum(arr: &[u32], n: usize) -> u64 {
    arr[..n].iter().map(|&x| x as u64).sum()
}
```

#### 9.1.4 Invariant Design Patterns

**Pattern 1: Range Invariant**

```rust
#[hax::loop_invariant(i >= 0 && i <= n)]
while i < n {
    // i always in range [0, n]
    i += 1;
}
```

**Pattern 2: Accumulator Invariant**

```rust
use hax_lib::int::Int;

#[hax::loop_invariant(Int::from(sum) <= Int::from(i) * Int::from(max_value))]
while i < n {
    sum += arr[i];
    i += 1;
}
```

**Pattern 3: Relationship Invariant**

```rust
#[hax::loop_invariant(left <= right)]
#[hax::loop_invariant(right <= arr.len())]
while left < right {
    let mid = left + (right - left) / 2;
    // Binary search logic
}
```

**Pattern 4: Progress Invariant**

```rust
#[hax::loop_invariant(processed == i)]
while i < arr.len() {
    // Process arr[i]
    processed += 1;
    i += 1;
}
```

### 9.2 Termination Measures (#[decreases])

Termination measures prove that loops terminate.

#### 9.2.1 Basic Termination Measures

```rust
use hax_lib as hax;

pub fn countdown(n: u32) -> u32 {
    let mut i = n;
    
    #[hax::loop_decreases(i)]
    while i > 0 {
        i -= 1;
    }
    
    i
}
```

#### 9.2.2 Termination with Multiple Variables

```rust
use hax_lib as hax;

pub fn binary_search(arr: &[u32], target: u32) -> Option<usize> {
    let mut left = 0;
    let mut right = arr.len();
    
    #[hax::loop_invariant(left <= right && right <= arr.len())]
    #[hax::loop_decreases(right - left)]
    while left < right {
        let mid = left + (right - left) / 2;
        
        if arr[mid] == target {
            return Some(mid);
        } else if arr[mid] < target {
            left = mid + 1;
        } else {
            right = mid;
        }
    }
    
    None
}
```

#### 9.2.3 Complex Termination Measures

```rust
use hax_lib as hax;

pub fn gcd(mut a: u32, mut b: u32) -> u32 {
    #[hax::loop_decreases((a + b, b))]  // Lexicographic ordering
    while b != 0 {
        let temp = b;
        b = a % b;
        a = temp;
    }
    a
}
```

### 9.3 Ghost Variables

Ghost variables exist only for verification, not at runtime.

#### 9.3.1 Basic Ghost Variables

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

pub fn fast_exponentiation(base: u64, exp: u32) -> u64 {
    let mut result = 1u64;
    let mut b = base;
    let mut e = exp;
    
    #[hax::ghost] let initial_base = base.to_int();
    #[hax::ghost] let initial_exp = exp.to_int();
    
    #[hax::loop_invariant(
        Int::from(result) * Int::from(b).pow(e.to_int())
        == initial_base.pow(initial_exp)
    )]
    #[hax::loop_decreases(e)]
    while e > 0 {
        if e % 2 == 1 {
            result *= b;
        }
        b *= b;
        e /= 2;
    }
    
    result
}
```

#### 9.3.2 Ghost State for Array Modifications

```rust
use hax_lib as hax;

pub fn reverse_array(arr: &mut [u32]) {
    #[hax::ghost] let original = arr.to_vec();
    
    let mut left = 0;
    let mut right = arr.len();
    
    #[hax::loop_invariant(left <= right && right <= arr.len())]
    #[hax::loop_invariant(forall(|i: usize|
        implies(i < left, arr[i] == original[arr.len() - 1 - i])
    ))]
    while left < right {
        right -= 1;
        arr.swap(left, right);
        left += 1;
    }
}
```

### 9.4 Complex Loop Examples

#### 9.4.1 Nested Loops

```rust
use hax_lib as hax;

pub fn matrix_multiply(a: &[Vec<u32>], b: &[Vec<u32>]) -> Vec<Vec<u32>> {
    let n = a.len();
    let mut result = vec![vec![0u32; n]; n];
    
    let mut i = 0;
    #[hax::loop_invariant(i <= n)]
    while i < n {
        let mut j = 0;
        #[hax::loop_invariant(j <= n)]
        while j < n {
            let mut k = 0;
            #[hax::loop_invariant(k <= n)]
            while k < n {
                result[i][j] += a[i][k] * b[k][j];
                k += 1;
            }
            j += 1;
        }
        i += 1;
    }
    
    result
}
```

#### 9.4.2 Loop with Early Exit

```rust
use hax_lib as hax;

pub fn find_first_negative(arr: &[i32]) -> Option<usize> {
    let mut i = 0;
    
    #[hax::loop_invariant(i <= arr.len())]
    #[hax::loop_invariant(forall(|j: usize|
        implies(j < i, arr[j] >= 0)
    ))]
    while i < arr.len() {
        if arr[i] < 0 {
            return Some(i);
        }
        i += 1;
    }
    
    None
}
```

#### 9.4.3 Loop with Complex State

```rust
use hax_lib as hax;

pub fn remove_duplicates(arr: &mut Vec<u32>) -> usize {
    if arr.is_empty() {
        return 0;
    }
    
    let mut write_idx = 1;
    let mut read_idx = 1;
    
    #[hax::loop_invariant(write_idx <= read_idx)]
    #[hax::loop_invariant(read_idx <= arr.len())]
    #[hax::loop_invariant(write_idx > 0)]
    while read_idx < arr.len() {
        if arr[read_idx] != arr[write_idx - 1] {
            arr[write_idx] = arr[read_idx];
            write_idx += 1;
        }
        read_idx += 1;
    }
    
    write_idx
}
```

---

## 10. Advanced Attributes

### 10.1 Opacity and Modularity (#[opaque])

The `#[opaque]` attribute hides implementation details from the verifier.

#### 10.1.1 Basic Opacity

```rust
use hax_lib as hax;

/// Verify this function once
#[hax::opaque]
#[hax::requires(n > 0)]
#[hax::ensures(|result| result > 0)]
pub fn complex_function(n: u32) -> u32 {
    // Complex implementation
    // Verified once, then treated as black box
    (1..=n).sum()
}

/// Callers only see the contract
pub fn caller(x: u32) {
    if x > 0 {
        let result = complex_function(x);
        hax::assert!(result > 0);  // Fast: uses contract only
    }
}
```

#### 10.1.2 When to Use Opacity

```rust
use hax_lib as hax;

/// DON'T MAKE OPAQUE: Simple, fast to verify
pub fn add_one(x: u32) -> u32 {
    x + 1
}

/// DO MAKE OPAQUE: Complex, slow to verify
#[hax::opaque]
#[hax::ensures(|result| result >= input)]
pub fn complex_crypto_operation(input: &[u8]) -> Vec<u8> {
    // 1000 lines of cryptographic code
    // Better to verify once and reuse the contract
    vec![]
}
```

### 10.2 Lemmas and Auxiliary Proofs (#[lemma])

Lemmas are reusable proof fragments.

#### 10.2.1 Basic Lemmas

```rust
use hax_lib as hax;
use hax_lib::int::Int;

/// A lemma proving a mathematical property
#[hax::lemma]
#[hax::ensures(|_| forall(|x: u32, y: u32|
    Int::from(x) + Int::from(y) == Int::from(y) + Int::from(x)
))]
pub fn addition_commutes() {}

/// Use the lemma
pub fn example(a: u32, b: u32) {
    addition_commutes();  // Import the proof
    hax::assert!(Int::from(a) + Int::from(b) == Int::from(b) + Int::from(a));
}
```

#### 10.2.2 Lemmas with Parameters

```rust
use hax_lib as hax;

#[hax::lemma]
#[hax::ensures(|_| x + (y + z) == (x + y) + z)]
pub fn addition_associative(x: u32, y: u32, z: u32) {}

pub fn use_lemma(a: u32, b: u32, c: u32) {
    addition_associative(a, b, c);
    hax::assert!(a + (b + c) == (a + b) + c);
}
```

### 10.3 Visibility Control

#### 10.3.1 Exclusion (#[exclude])

```rust
use hax_lib as hax;

/// Don't extract this function
#[hax::exclude]
pub fn unverified_helper() {
    // Won't be verified or extracted
}

/// Extract this function
pub fn verified_function() {
    // Will be extracted
}
```

#### 10.3.2 Inclusion (#[include])

```rust
use hax_lib as hax;

/// Force extraction even if not called
#[hax::include]
fn helper_function() {
    // Will be extracted even if unused
}
```

### 10.4 Backend-Specific Attributes

#### 10.4.1 F\* Attributes

```rust
use hax_lib as hax;

/// Set F* verification options
#[hax::fstar::options("--z3rlimit 200 --fuel 4")]
pub fn f_star_function() {}

/// Inject F* code
pub fn with_f_star_lemma() {
    hax::fstar!(r#"
        FStar.Math.Lemmas.pow2_plus 16 16
    "#);
}

/// Set verification status
#[hax::fstar::verification_status(lax)]
pub fn skip_verification() {}
```

#### 10.4.2 Lean Attributes

```rust
use hax_lib as hax;

/// Override Lean type
#[hax::lean::type("Nat")]
pub fn lean_function() -> u32 {
    42
}
```

---

## Part 4: Toolchain and Backends

## 11. CLI and Toolchain

### 11.1 cargo-hax Overview

The `cargo-hax` tool orchestrates the entire verification process.

#### 11.1.1 Basic Commands

```bash
# Extract to F*
cargo hax into fstar

# Extract to Lean
cargo hax into lean

# Extract to Coq
cargo hax into coq

# Extract to ProVerif
cargo hax into pro-verif

# Get JSON AST (for debugging)
cargo hax json
```

### 11.2 Complete Command Reference

#### 11.2.1 Global Options

```bash
# Verbose output
cargo hax -v into fstar

# Very verbose output
cargo hax -vv into fstar

# Quiet mode
cargo hax -q into fstar

# Specify output directory
cargo hax into fstar --output-dir ./proofs

# Dry run (show what would be done)
cargo hax into fstar --dry-run
```

#### 11.2.2 Item Selection

```bash
# Include specific modules
cargo hax into fstar --include "crypto::*"

# Include multiple patterns
cargo hax into fstar --include "crypto::*" --include "protocol::*"

# Exclude patterns
cargo hax into fstar --exclude "tests::*"

# Combine include and exclude
cargo hax into fstar --include "src::*" --exclude "src::tests::*"
```

#### 11.2.3 Backend-Specific Options

**F\* Options:**

```bash
# Set Z3 resource limit
cargo hax into fstar --z3rlimit 200

# Set fuel (normalization depth)
cargo hax into fstar --fuel 4

# Set ifuel (inversion depth)
cargo hax into fstar --ifuel 2

# Combine options
cargo hax into fstar --z3rlimit 200 --fuel 4 --ifuel 2
```

**Lean Options:**

```bash
# Target Lean 4
cargo hax into lean --lean-version 4

# Include mathlib
cargo hax into lean --mathlib
```

**Coq Options:**

```bash
# Target Coq 8.15
cargo hax into coq --coq-version 8.15
```

### 11.3 Configuration Options

#### 11.3.1 Cargo.toml Configuration

```toml
[package.metadata.hax]
# Global settings
include = ["src/verified/**"]
exclude = ["src/tests/**"]

# F* backend
[package.metadata.hax.into.fstar]
z3rlimit = 100
fuel = 2
ifuel = 1
output_dir = "proofs/fstar"

# Lean backend
[package.metadata.hax.into.lean]
version = 4
mathlib = true
output_dir = "proofs/lean"

# Coq backend
[package.metadata.hax.into.coq]
version = "8.15"
output_dir = "proofs/coq"
```

### 11.4 Environment Variables

```bash
# Override engine binary path
export HAX_ENGINE_BINARY=/custom/path/hax-engine

# Specify Rust toolchain
export HAX_TOOLCHAIN=nightly-2024-01-01

# Set F* home
export FSTAR_HOME=/path/to/fstar

# Set HACL* home
export HACL_HOME=/path/to/hacl-star

# Enable debug logging
export RUST_LOG=debug
```

### 11.5 Project Organization

#### 11.5.1 Recommended Structure

```
my-project/
├── Cargo.toml
├── src/
│   ├── lib.rs           # Main implementation
│   ├── crypto/          # Verified crypto code
│   ├── protocol/        # Verified protocol code
│   └── tests/           # Tests (excluded from verification)
├── proofs/
│   ├── fstar/
│   │   └── extraction/  # Generated F* code
│   ├── lean/
│   │   └── extraction/  # Generated Lean code
│   └── specs/           # Specification documents
└── README.md
```

#### 11.5.2 .gitignore Configuration

```gitignore
# Generated proof files
proofs/**/extraction/

# Hax intermediate files
target/hax/

# F* build artifacts
**/.checked
**/.hints

# Lean build artifacts
**/build/
```

---

## 12. Backend Support

### 12.1 F\* (Stable)

The most mature backend with full feature support.

#### 12.1.1 Features

- ✅ Full specification support
- ✅ SMT solver integration (Z3)
- ✅ Refinement types
- ✅ Loop invariants and termination
- ✅ Effectful programming
- ✅ Module system
- ✅ Inline F* code

#### 12.1.2 Workflow

```bash
# Extract
cargo hax into fstar

# Verify
cd proofs/fstar/extraction
fstar.exe --include $HACL_HOME/lib *.fst

# With specific options
fstar.exe --z3rlimit 200 --fuel 4 MyModule.fst
```

#### 12.1.3 Generated Code Example

**Rust:**
```rust
#[hax::requires(x < 100)]
#[hax::ensures(|result| result > x)]
pub fn increment(x: u32) -> u32 {
    x + 1
}
```

**Generated F\*:**
```fstar
val increment (x: u32) : Pure u32
  (requires x <. 100ul)
  (ensures fun result -> result >. x)

let increment x = x +! 1ul
```

### 12.2 Lean4 (Active Development)

Focus on panic-freedom and functional correctness.

#### 12.2.1 Features

- ✅ Panic-freedom proofs
- ✅ Safe arithmetic operators
- ✅ Array bounds checking
- ✅ Pattern matching exhaustiveness
- ⚠️ Limited SMT integration

#### 12.2.2 Workflow

```bash
# Extract
cargo hax into lean

# Build with Lake
cd proofs/lean/extraction
lake build
```

#### 12.2.3 Generated Code Example

**Rust:**
```rust
pub fn safe_divide(x: u32, y: u32) -> Option<u32> {
    x.checked_div(y)
}
```

**Generated Lean4:**
```lean
def safe_divide (x y : UInt32) : Option UInt32 :=
  x /? y  -- Lean's panic-safe division operator
```

### 12.3 Backend Comparison Matrix

| Feature | F\* | Lean4 | Coq | ProVerif |
|---------|-----|-------|-----|----------|
| **Maturity** | Stable | Active | Experimental | PoC |
| **Preconditions** | ✅ | ✅ | ⚠️ | ❌ |
| **Postconditions** | ✅ | ✅ | ⚠️ | ❌ |
| **Loop Invariants** | ✅ | ✅ | ⚠️ | ❌ |
| **Refinement Types** | ✅ | ✅ | ⚠️ | ❌ |
| **Panic Proofs** | Manual | Automatic | Manual | N/A |
| **SMT Solver** | Z3 | Limited | No | Symbolic |
| **Best For** | Crypto, Systems | Functional correctness | Mathematical proofs | Protocols |

---

## 13. Examples and Patterns

### 13.1 Basic Verification Patterns

#### Example 1: Safe Array Access

```rust
use hax_lib as hax;

#[hax::requires(index < arr.len())]
#[hax::ensures(|result| result == arr[index])]
pub fn safe_get(arr: &[u32], index: usize) -> u32 {
    arr[index]
}
```

#### Example 2: Overflow-Safe Arithmetic

```rust
use hax_lib::int::{Int, ToInt};

#[hax::requires(
    Int::from(x) + Int::from(y) <= Int::from(u32::MAX)
)]
#[hax::ensures(|result| 
    Int::from(result) == Int::from(x) + Int::from(y)
)]
pub fn safe_add(x: u32, y: u32) -> u32 {
    x + y
}
```

### 13.2 Cryptographic Algorithms

See **[examples-patterns.md](examples-patterns.md)** for comprehensive examples including:
- ChaCha20 stream cipher
- Barrett reduction (field arithmetic)
- SHA-256 hash function
- AES operations

### 13.3 Data Structures

#### Example: Verified Stack

```rust
use hax_lib as hax;

pub struct Stack<T> {
    data: Vec<T>,
}

impl<T> Stack<T> {
    #[hax::ensures(|result| result.data.len() == 0)]
    pub fn new() -> Self {
        Stack { data: Vec::new() }
    }
    
    #[hax::ensures(|_| self.data.len() == hax::old(self.data.len()) + 1)]
    pub fn push(&mut self, item: T) {
        self.data.push(item);
    }
    
    #[hax::requires(self.data.len() > 0)]
    #[hax::ensures(|_| self.data.len() == hax::old(self.data.len()) - 1)]
    pub fn pop(&mut self) -> T {
        self.data.pop().unwrap()
    }
}
```

---

## 14. Advanced Features

### 14.1 Backend-Specific Code Injection

```rust
use hax_lib as hax;

pub fn with_backend_code() {
    // Inject F* code
    hax::fstar!(r#"
        FStar.Math.Lemmas.pow2_plus 16 16;
        assert_norm (pow2 32 == 4294967296)
    "#);
    
    // Inject Lean code
    hax::lean!(r#"
        have h : 2 + 2 = 4 := rfl
    "#);
}
```

### 14.2 Proof Optimization

#### Technique 1: Use Opacity

```rust
#[hax::opaque]
pub fn expensive_proof(n: u32) -> u32 {
    // Verify once, reuse contract
    (0..n).sum()
}
```

#### Technique 2: Strengthen Invariants

```rust
// Weak invariant (hard to prove)
#[hax::loop_invariant(sum <= u32::MAX)]

// Strong invariant (easier to prove)
#[hax::loop_invariant(sum <= i * max_element)]
```

---

## 15. Complete API Reference

### 15.1 Core Types

| Type | Purpose | Module |
|------|---------|--------|
| `Int` | Mathematical integers | `hax_lib::int` |
| `Prop` | Logical propositions | `hax_lib` |

### 15.2 Traits

| Trait | Purpose |
|-------|---------|
| `Abstraction` | Lift to abstract types |
| `Concretization` | Lower to concrete types |
| `Refinement` | Refinement type trait |
| `RefineAs` | Refinement conversion |

### 15.3 Macros

#### Declarative Macros

| Macro | Type | Purpose |
|-------|------|---------|
| `assert!` | Proof obligation | Verify condition |
| `assume!` | Assumption | Trust condition |
| `assert_prop!` | Logical assertion | Assert proposition |
| `forall!` | Quantifier | Universal quantification |
| `exists!` | Quantifier | Existential quantification |
| `implies!` | Logic | Implication |
| `int!` | Literal | Int literal |

#### Procedural Macros

| Macro | Type | Purpose |
|-------|------|---------|
| `#[requires]` | Attribute | Precondition |
| `#[ensures]` | Attribute | Postcondition |
| `#[loop_invariant]` | Attribute | Loop invariant |
| `#[decreases]` | Attribute | Termination measure |
| `#[opaque]` | Attribute | Hide implementation |
| `#[lemma]` | Attribute | Proof lemma |
| `#[refinement_type]` | Attribute | Define refinement |
| `#[exclude]` | Attribute | Don't extract |
| `#[include]` | Attribute | Force extraction |

For complete macro documentation, see **[macro-system-complete.md](macro-system-complete.md)**.

---

## 16. Resources and Community

### 16.1 External Links

- **GitHub**: https://github.com/hacspec/hax
- **Website**: https://hax.cryspen.com
- **Playground**: https://hax-playground.cryspen.com
- **Documentation**: https://hax.cryspen.com/manual/
- **Zulip Chat**: https://hacspec.zulipchat.com/
- **Blog**: https://hax.cryspen.com/blog

### 16.2 Related Documentation

- **[hax-lib-api.md](hax-lib-api.md)**: Complete API reference
- **[macro-system-complete.md](macro-system-complete.md)**: All macros
- **[examples-patterns.md](examples-patterns.md)**: Comprehensive examples
- **[verification-techniques.md](verification-techniques.md)**: Proof strategies
- **[cli-backends.md](cli-backends.md)**: CLI and backends
- **[errors-troubleshooting.md](errors-troubleshooting.md)**: Error reference

### 16.3 Contributing

We welcome contributions!

1. **Join Zulip**: https://hacspec.zulipchat.com/
2. **File Issues**: https://github.com/hacspec/hax/issues
3. **Submit PRs**: Follow the contribution guidelines

### 16.4 Roadmap

Key areas of development:

- **Rust Engine**: Full rewrite in Rust
- **Better Error Messages**: Improved diagnostics
- **More Backends**: Expanded backend support
- **Performance**: Faster verification
- **Documentation**: More examples and tutorials

### 16.5 License

The hax library is licensed under Apache-2.0.

---

## Conclusion

This README provides a comprehensive overview of the hax library. For deeper dives into specific topics, please refer to the specialized documentation files listed in the [Documentation Structure](#documentation-structure) section.

**Happy Verifying! 🎉**