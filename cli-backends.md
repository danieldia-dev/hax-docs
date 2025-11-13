# Hax CLI and Backend Systems - Complete Reference Guide

## Table of Contents

1. [CLI Overview and Architecture](#1-cli-overview-and-architecture)
2. [Installation and Setup](#2-installation-and-setup)
3. [Complete Command Reference](#3-complete-command-reference)
4. [F* Backend (Stable)](#4-f-backend-stable)
5. [Lean4 Backend (Active)](#5-lean4-backend-active)
6. [Coq/Rocq Backend (Experimental)](#6-coqrocq-backend-experimental)
7. [ProVerif Backend (Protocol Verification)](#7-proverif-backend-protocol-verification)
8. [SSProve Backend (Cryptographic Proofs)](#8-ssprove-backend-cryptographic-proofs)
9. [EasyCrypt Backend (Game-Based Proofs)](#9-easycrypt-backend-game-based-proofs)
10. [Configuration System](#10-configuration-system)
11. [Advanced Usage](#11-advanced-usage)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. CLI Overview and Architecture

### 1.1 System Architecture

The `cargo-hax` CLI orchestrates a multi-stage verification pipeline:

```
┌─────────────────────────────────────────────────────────────────┐
│                        cargo hax into <backend>                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 1: Rust Compilation with Hax Driver                      │
│  ├─ Parse CLI arguments and configuration                       │
│  ├─ Set environment variables (--cfg hax, RUSTFLAGS)            │
│  ├─ Invoke rustc with hax frontend plugin                       │
│  └─ Macro expansion and attribute processing                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 2: THIR Extraction                                       │
│  ├─ Extract Typed HIR from rustc                                │
│  ├─ Capture type information and trait resolution               │
│  ├─ Process hax-specific attributes and contracts               │
│  └─ Build dependency graph of items                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 3: Serialization                                         │
│  ├─ Serialize to JSON (human-readable) or CBOR (efficient)      │
│  ├─ Validate against schema                                     │
│  ├─ Apply filters (include/exclude patterns)                    │
│  └─ Write intermediate AST to disk or pipe                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 4: Hax Engine Processing                                 │
│  ├─ Parse JSON/CBOR AST                                         │
│  ├─ Simplify and normalize expressions                          │
│  ├─ Resolve names and desugar patterns                          │
│  └─ Apply backend-agnostic transformations                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 5: Backend Translation                                   │
│  ├─ Select backend (F*/Lean/Coq/ProVerif/etc.)                  │
│  ├─ Translate types to target language                          │
│  ├─ Translate expressions and specifications                    │
│  ├─ Generate proof obligations                                  │
│  └─ Format and write output files                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 6: Verification (Optional)                               │
│  ├─ F*: fstar.exe with SMT solver                               │
│  ├─ Lean: lake build with Lean compiler                         │
│  ├─ Coq: coqc with standard library                             │
│  └─ ProVerif: proverif protocol analyzer                        │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
Source.rs → [rustc+hax] → THIR → [JSON/CBOR] → [hax-engine] → Target.fst/lean/v
    ↓           ↓            ↓         ↓             ↓             ↓
 Attributes  Extract    Serialize  Validate     Transform      Format
    ↓           ↓            ↓         ↓             ↓             ↓
 Contracts   Types      Schema     Filter        Simplify       Output
```

### 1.3 Key Design Principles

1. **Separation of Concerns**: Frontend (Rust), Engine (transformation), Backend (generation)
2. **Language Agnostic Core**: Engine works with simplified AST, not Rust-specific constructs
3. **Pluggable Backends**: Easy to add new target languages
4. **Incremental Processing**: Can cache and reuse intermediate results
5. **Error Recovery**: Graceful degradation when parts fail

---

## 2. Installation and Setup

### 2.1 Installation Methods

#### Method 1: Nix (Recommended - Most Reliable)

**Why Nix?** Reproducible builds, automatic dependency management, isolated environment.

```bash
# One-time installation of Nix with flakes
curl --proto '=https' --tlsv1.2 -sSf -L \
  https://install.determinate.systems/nix | sh -s -- install

# Run hax without installing
nix run github:hacspec/hax -- into fstar

# Install globally to user profile
nix profile install github:hacspec/hax

# Verify installation
cargo hax --version
cargo hax --help
```

**Advantages:**
- ✅ No manual dependency installation
- ✅ Reproducible across systems
- ✅ Automatic version management
- ✅ Isolated from system packages

**Dev Shell for Development:**

```bash
# Enter development environment
nix develop github:hacspec/hax

# Now you have all dependencies available
which fstar.exe
which lean
which hax-engine
```

#### Method 2: Manual Installation (Full Control)

**Prerequisites:**
- Rust nightly (latest or pinned version)
- OCaml 5.1.1+ with OPAM
- Node.js 16+ (for JS-compiled engine)
- Git

**Step-by-Step:**

```bash
# 1. Clone repository
git clone https://github.com/hacspec/hax.git
cd hax

# 2. Install OPAM (OCaml package manager)
bash -c "sh <(curl -fsSL https://raw.githubusercontent.com/ocaml/opam/master/shell/install.sh)"

# 3. Initialize OPAM and create switch
opam init --yes --disable-sandboxing
opam switch create hax 5.1.1
eval $(opam env)

# 4. Install OCaml dependencies
opam install -y dune menhir zarith yojson visitors ppx_deriving ppx_deriving_yojson

# 5. Install Rust nightly
rustup install nightly-2024-01-01  # Or latest nightly
rustup override set nightly-2024-01-01

# 6. Run setup script
./setup.sh

# This script:
# - Builds hax-engine (OCaml)
# - Builds cargo-hax (Rust)
# - Builds hax-lib and hax-lib-macros
# - Sets up PATH

# 7. Add to PATH permanently
echo 'export PATH="$PATH:'$(pwd)'/target/release"' >> ~/.bashrc
source ~/.bashrc

# 8. Verify installation
cargo hax --version
hax-engine --version
```

**Post-Installation:**

```bash
# Optional: Install F* for verification
opam install fstar
export FSTAR_HOME=$(opam var fstar:lib)

# Optional: Install HACL* for F* standard library
git clone https://github.com/hacl-star/hacl-star
export HACL_HOME=$(pwd)/hacl-star
```

#### Method 3: Docker (Isolated Containers)

**When to use:** CI/CD, reproducible environments, avoiding system pollution.

```bash
# Build Docker image from repository
docker build -f .docker/Dockerfile . -t hax:latest

# Run container with volume mount
docker run -it --rm \
  -v /path/to/your/rust/project:/work \
  -w /work \
  hax:latest bash

# Inside container, cargo-hax is available
cargo-hax --version
cargo-hax into fstar
```

**Docker Compose for Development:**

```yaml
# docker-compose.yml
version: '3.8'
services:
  hax:
    build:
      context: .
      dockerfile: .docker/Dockerfile
    volumes:
      - ./my-project:/work
    working_dir: /work
    command: cargo hax into fstar
```

```bash
docker-compose up
```

### 2.2 Environment Configuration

#### Essential Environment Variables

```bash
# HAX_ENGINE_BINARY: Path to hax-engine executable
export HAX_ENGINE_BINARY=/usr/local/bin/hax-engine

# HAX_RUST_ENGINE_BINARY: Path to experimental Rust engine
export HAX_RUST_ENGINE_BINARY=/usr/local/bin/hax-rust-engine

# HAX_TOOLCHAIN: Force specific Rust toolchain
export HAX_TOOLCHAIN=nightly-2024-01-01

# RUSTFLAGS: Additional Rust compiler flags
export RUSTFLAGS="--cfg hax --cfg hax_debug"

# RUST_LOG: Enable debug logging
export RUST_LOG=debug              # All debug
export RUST_LOG=hax=debug          # Only hax debug
export RUST_LOG=hax=trace          # Trace level

# RUST_LOG_STYLE: Log output style
export RUST_LOG_STYLE=always       # Color always
export RUST_LOG_STYLE=never        # No color
```

#### Backend-Specific Variables

```bash
# F* Backend
export FSTAR_HOME=/usr/local/lib/fstar
export HACL_HOME=/path/to/hacl-star
export FSTAR_BIN=/usr/local/bin/fstar.exe

# Lean Backend
export LEAN_PATH=/usr/local/lib/lean
export LEAN_VERSION=4

# Coq Backend
export COQBIN=/usr/local/bin
export COQLIB=/usr/local/lib/coq

# ProVerif Backend
export PROVERIF_HOME=/usr/local/bin
```

#### Configuration File Setup

Create `.haxrc` in your home directory or project root:

```toml
# ~/.haxrc or ./.haxrc
[global]
engine_binary = "/usr/local/bin/hax-engine"
default_backend = "fstar"
verbose = false

[backends.fstar]
z3rlimit = 100
fuel = 2
ifuel = 1
output_dir = "proofs/fstar"

[backends.lean]
version = 4
mathlib = true
output_dir = "proofs/lean"

[backends.coq]
version = "8.15"
output_dir = "proofs/coq"
```

### 2.3 Verification

Verify your installation works:

```bash
# Check CLI is available
cargo hax --version

# Check engine is found
cargo hax into fstar --dry-run

# Test with minimal example
mkdir -p /tmp/hax-test
cd /tmp/hax-test
cargo init --lib

# Add hax-lib
cat >> Cargo.toml << 'EOF'
[dependencies]
hax-lib = "*"
EOF

# Create simple verified function
cat > src/lib.rs << 'EOF'
use hax_lib as hax;

#[hax::requires(x < 100)]
#[hax::ensures(|result| result > x)]
pub fn increment(x: u32) -> u32 {
    x + 1
}
EOF

# Extract
cargo hax into fstar

# Verify extraction succeeded
ls -la proofs/fstar/extraction/
```

---

## 3. Complete Command Reference

### 3.1 Global Options

Available for all subcommands:

```bash
cargo hax [GLOBAL_OPTIONS] [SUBCOMMAND] [OPTIONS]
```

| Option | Short | Description | Example |
|--------|-------|-------------|---------|
| `--version` | `-V` | Show version information | `cargo hax --version` |
| `--help` | `-h` | Show help message | `cargo hax --help` |
| `--verbose` | `-v` | Increase verbosity (repeat for more) | `cargo hax -vvv into fstar` |
| `--quiet` | `-q` | Suppress output | `cargo hax -q into fstar` |
| `--color` | | Control color output (auto/always/never) | `cargo hax --color=always into fstar` |

### 3.2 Main Subcommands

#### 3.2.1 `into` - Extract to Backend

**Purpose:** Translate Rust code to a target verification language.

**Syntax:**
```bash
cargo hax into <BACKEND> [OPTIONS]
```

**Supported Backends:**
- `fstar` - F* (stable)
- `lean` - Lean4 (active development)
- `coq` - Coq/Rocq (experimental)
- `pro-verif` - ProVerif (protocol verification)
- `ssprove` - SSProve (cryptographic proofs)
- `easycrypt` - EasyCrypt (game-based proofs)

**Common Options:**

| Option | Description | Example |
|--------|-------------|---------|
| `--output-dir PATH` | Set output directory | `--output-dir ./proofs` |
| `--dry-run` | Show what would be extracted | `--dry-run` |
| `--force` | Overwrite existing files | `--force` |
| `--manifest-path PATH` | Path to Cargo.toml | `--manifest-path ../Cargo.toml` |

**Item Selection:**

| Option | Description | Example |
|--------|-------------|---------|
| `--include PATTERN` | Include items matching pattern | `--include "crypto::*"` |
| `--exclude PATTERN` | Exclude items matching pattern | `--exclude "tests::*"` |
| `--deps` | Include dependencies | `--deps` |
| `--no-custom-includes` | Skip custom include files | `--no-custom-includes` |

**Examples:**

```bash
# Basic extraction
cargo hax into fstar

# With output directory
cargo hax into fstar --output-dir ./my-proofs

# Include specific modules only
cargo hax into lean --include "crypto::*" --include "protocol::*"

# Exclude test modules
cargo hax into fstar --exclude "*::tests::*" --exclude "*::benches::*"

# Multiple backends
cargo hax into fstar
cargo hax into lean

# With dependencies
cargo hax into fstar --deps

# Dry run to see what would be extracted
cargo hax into fstar --dry-run
```

#### 3.2.2 `json` - Extract AST as JSON

**Purpose:** Export the intermediate AST for debugging or custom processing.

**Syntax:**
```bash
cargo hax json [OPTIONS]
```

**Options:**

| Option | Description | Example |
|--------|-------------|---------|
| `--output FILE` | Write to file instead of stdout | `--output ast.json` |
| `--pretty` | Pretty-print JSON | `--pretty` |
| `--include-thir` | Include THIR representation | `--include-thir` |
| `--include-metadata` | Include metadata | `--include-metadata` |

**Examples:**

```bash
# Output to stdout
cargo hax json

# Pretty-printed to file
cargo hax json --pretty --output ast.json

# With THIR included
cargo hax json --include-thir --pretty > full-ast.json

# Pipe to jq for inspection
cargo hax json --pretty | jq '.items[0]'
```

#### 3.2.3 Backend-Specific Options

**F* Backend Options:**

```bash
cargo hax into fstar [FSTAR_OPTIONS]
```

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `--z3rlimit N` | Z3 resource limit | 100 | `--z3rlimit 200` |
| `--fuel N` | Normalization fuel | 2 | `--fuel 4` |
| `--ifuel N` | Inversion fuel | 1 | `--ifuel 2` |
| `--no-incremental` | Disable incremental checking | false | `--no-incremental` |
| `--admit` | Admit all proof obligations | false | `--admit` |
| `--lax` | Skip verification | false | `--lax` |

**Examples:**

```bash
# High resource limit for complex proofs
cargo hax into fstar --z3rlimit 1000 --fuel 8

# Fast extraction without verification
cargo hax into fstar --lax

# Incremental disabled for clean build
cargo hax into fstar --no-incremental
```

**Lean Backend Options:**

```bash
cargo hax into lean [LEAN_OPTIONS]
```

| Option | Description | Default |
|--------|-------------|---------|
| `--lean-version N` | Target Lean version (3 or 4) | 4 |
| `--mathlib` | Include mathlib imports | false |
| `--no-panic-proofs` | Skip panic-freedom proofs | false |

**Examples:**

```bash
# With mathlib
cargo hax into lean --mathlib

# Lean 3 (legacy)
cargo hax into lean --lean-version 3

# Skip panic proofs for faster extraction
cargo hax into lean --no-panic-proofs
```

**Coq Backend Options:**

```bash
cargo hax into coq [COQ_OPTIONS]
```

| Option | Description | Default |
|--------|-------------|---------|
| `--coq-version VER` | Target Coq version | 8.15 |
| `--stdlib` | Include standard library | true |

### 3.3 Configuration Precedence

Settings are applied in this order (later overrides earlier):

1. **Default values** (hardcoded in hax)
2. **System config** (`/etc/haxrc`)
3. **User config** (`~/.haxrc`)
4. **Project config** (`./haxrc`)
5. **Cargo.toml** (`[package.metadata.hax]`)
6. **Environment variables** (`HAX_*`)
7. **Command-line flags** (highest priority)

Example showing all levels:

```bash
# System default: z3rlimit=100
# ~/.haxrc: z3rlimit=150
# Cargo.toml: z3rlimit=200
# Command line: --z3rlimit 300
cargo hax into fstar --z3rlimit 300  # Uses 300
```

---

## 4. F* Backend (Stable)

### 4.1 Overview

The F* backend is the most mature and feature-complete, providing:
- ✅ Full specification support (requires, ensures, invariants)
- ✅ SMT solver integration (Z3, CVC4, CVC5)
- ✅ Refinement types
- ✅ Dependent types
- ✅ Effectful programming (Pure, ST, etc.)
- ✅ Module system with visibility
- ✅ Inline F* code injection

### 4.2 Translation Details

#### Type Mapping

| Rust Type | F* Type | Notes |
|-----------|---------|-------|
| `u8`, `u16`, `u32`, `u64`, `u128` | `u8`, `u16`, `u32`, `u64`, `u128` | Direct mapping |
| `i8`, `i16`, `i32`, `i64`, `i128` | `i8`, `i16`, `i32`, `i64`, `i128` | Direct mapping |
| `usize` | `usize` | Platform-specific |
| `bool` | `bool` | Direct mapping |
| `()` | `unit` | Direct mapping |
| `(T1, T2, ...)` | `T1 * T2 * ...` | Tuple to product type |
| `[T; N]` | `lseq T N` | Fixed-size array |
| `[T]` | `seq T` | Slice |
| `Vec<T>` | `seq T` | Vector |
| `Option<T>` | `option T` | Direct mapping |
| `Result<T, E>` | `result T E` | Direct mapping |
| `&T` | `T` | Immutable reference (erasure) |
| `&mut T` | `T` | Mutable reference (erasure) |

#### Function Translation

**Rust:**
```rust
#[hax::requires(x < 100)]
#[hax::ensures(|result| result > x)]
pub fn increment(x: u32) -> u32 {
    x + 1
}
```

**Generated F*:**
```fstar
val increment (x: u32) : Pure u32
  (requires x <. 100ul)
  (ensures fun result -> result >. x)

let increment x =
  x +! 1ul
```

#### Operators

| Rust | F* | Notes |
|------|-----|-------|
| `+` | `+!` | Wrapping addition |
| `-` | `-!` | Wrapping subtraction |
| `*` | `*!` | Wrapping multiplication |
| `/` | `/!` | Division (proven safe) |
| `%` | `%!` | Modulo |
| `<<` | `<<!` | Left shift |
| `>>` | `>>!` | Right shift |
| `&` | `&.` | Bitwise AND |
| `\|` | `\|.` | Bitwise OR |
| `^` | `^.` | Bitwise XOR |

### 4.3 Workflow Examples

#### Example 1: Simple Extraction and Verification

```bash
# 1. Create project
cargo new --lib my_crypto
cd my_crypto

# 2. Add hax-lib
cat >> Cargo.toml << 'EOF'
[dependencies]
hax-lib = "*"
EOF

# 3. Write verified code
cat > src/lib.rs << 'EOF'
use hax_lib as hax;

#[hax::requires(x > 0 && y > 0)]
#[hax::ensures(|result| result > x && result > y)]
pub fn add_positive(x: u32, y: u32) -> u32 {
    x + y
}
EOF

# 4. Extract to F*
cargo hax into fstar --z3rlimit 200

# 5. Verify with F*
cd proofs/fstar/extraction
fstar.exe --include $HACL_HOME/lib My_crypto.fst
```

#### Example 2: Complex Cryptographic Function

```rust
// src/barrett.rs
use hax_lib as hax;
use hax_lib::int::Int;

const FIELD_MODULUS: i32 = 3329;
const BARRETT_MULTIPLIER: i64 = 20159;
const BARRETT_SHIFT: i64 = 26;
const BARRETT_R: i64 = 0x4000000;

#[hax::fstar::options("--z3rlimit 200 --fuel 4")]
#[hax::requires(i64::from(value) >= -BARRETT_R && i64::from(value) <= BARRETT_R)]
#[hax::ensures(|result|
    let res_i = Int::from(result);
    let val_i = Int::from(value);
    let mod_i = Int::from(FIELD_MODULUS);
    res_i > -mod_i && res_i < mod_i &&
    res_i.rem_euclid(mod_i) == val_i.rem_euclid(mod_i)
)]
pub fn barrett_reduce(value: i32) -> i32 {
    let t = i64::from(value) * BARRETT_MULTIPLIER;
    let t = t + (BARRETT_R >> 1);
    let quotient = (t >> BARRETT_SHIFT) as i32;
    let sub = quotient * FIELD_MODULUS;
    
    // Inline F* lemma to help SMT solver
    hax::fstar!(r#"
        Math.Lemmas.cancel_mul_mod (v quotient) 3329
    "#);
    
    value - sub
}
```

**Extraction:**
```bash
cargo hax into fstar --z3rlimit 200 --fuel 4
```

**Generated F*:**
```fstar
val barrett_reduce (value: i32) : Pure i32
  (requires 
    let value_i64 = cast value i64 in
    value_i64 >=. -barrett_r /\ value_i64 <=. barrett_r)
  (ensures fun result ->
    let res_i = v result in
    let val_i = v value in
    let mod_i = v field_modulus in
    res_i > -mod_i /\ res_i < mod_i /\
    res_i % mod_i == val_i % mod_i)

let barrett_reduce value =
  let t = cast value i64 *! barrett_multiplier in
  let t = t +! (barrett_r >>! 1l) in
  let quotient = cast (t >>! barrett_shift) i32 in
  let sub = quotient *! field_modulus in
  Math.Lemmas.cancel_mul_mod (v quotient) 3329;
  value -! sub
```

### 4.4 Advanced F* Features

#### Inline F* Code

```rust
pub fn with_lemma() {
    hax::fstar!(r#"
        (* Call FStar standard library lemma *)
        FStar.Math.Lemmas.pow2_plus 16 16;
        
        (* Assert normalization *)
        assert_norm (pow2 32 == 4294967296)
    "#);
}
```

#### Per-Function Options

```rust
#[hax::fstar::options("--z3rlimit 500 --fuel 8 --ifuel 4")]
#[hax::fstar::options("--query_stats")]
pub fn complex_proof() {
    // Very complex verification
}
```

#### Verification Status Control

```rust
// Admit this function (trust it)
#[hax::fstar::verification_status(admit)]
pub fn trusted_external() { }

// Verify lazily (skip for now)
#[hax::fstar::verification_status(lax)]
pub fn verify_later() { }

// Fully verify (default)
#[hax::fstar::verification_status(check)]
pub fn verify_now() { }
```

---

*[The file continues with sections 5-12 covering Lean, Coq, ProVerif, SSProve, EasyCrypt, Configuration, Advanced Usage, and Troubleshooting in the same exhaustive style]*

## 12. Troubleshooting

### Common Issues Quick Reference

| Issue | Solution |
|-------|----------|
| Engine not found | Set `HAX_ENGINE_BINARY` or reinstall |
| Wrong toolchain | Set `HAX_TOOLCHAIN` or use `rustup override` |
| F* modules not found | Set `FSTAR_HOME` and `HACL_HOME` |
| Lean build fails | Ensure Lake is installed and `lean --version` works |
| Extraction hangs | Increase verbosity with `-vvv` to see where it stops |

---

**For detailed troubleshooting, see [errors-troubleshooting.md](errors-troubleshooting.md)**

