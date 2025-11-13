# Hax Error Messages and Troubleshooting - Complete Guide

## Table of Contents

1. [Error Categories](#error-categories)
2. [Compilation Errors](#compilation-errors)
3. [Verification Failures](#verification-failures)
4. [Backend-Specific Errors](#backend-specific-errors)
5. [Type System Errors](#type-system-errors)
6. [Macro Expansion Errors](#macro-expansion-errors)
7. [Engine Errors](#engine-errors)
8. [Performance Issues](#performance-issues)
9. [Environment and Setup Issues](#environment-and-setup-issues)
10. [Debugging Strategies](#debugging-strategies)
11. [Common Solutions](#common-solutions)
12. [Error Code Reference](#error-code-reference)

---

## Error Categories

`hax` errors fall into several distinct categories, each requiring different troubleshooting approaches:

| Category | Error Code Range | Description |
|----------|-----------------|-------------|
| Compilation | E0001-E0999 | Rust compilation and syntax errors |
| Verification | E1000-E1999 | Proof obligation failures |
| Backend | E2000-E2999 | Target language generation errors |
| Type System | E3000-E3999 | Type checking and inference errors |
| Macro | E4000-E4999 | Macro expansion failures |
| Engine | E5000-E5999 | Translation engine errors |
| Environment | E6000-E6999 | Setup and configuration issues |
| Internal | E9000-E9999 | Internal compiler errors |

---

## Compilation Errors

### Error: `trait bound not satisfied for hax_lib`

**Error Message:**
```
error[E0277]: the trait bound `T: hax_lib::Abstraction` is not satisfied
  --> src/lib.rs:10:5
   |
10 |     x.lift()
   |       ^^^^ the trait `hax_lib::Abstraction` is not implemented for `T`
```

**Cause:** Type `T` doesn't implement the required abstraction trait.

**Solutions:**
```rust
// Solution 1: Add trait bound
fn my_function<T: hax_lib::Abstraction>(x: T) -> T::AbstractType {
    x.lift()
}

// Solution 2: Implement the trait
impl Abstraction for MyType {
    type AbstractType = Int;
    fn lift(self) -> Self::AbstractType {
        Int::from(self.value)
    }
}

// Solution 3: Use a concrete type that already implements it
fn my_function(x: u32) -> Int {
    x.lift()  // u32 already implements Abstraction
}
```

### Error: `cfg(hax) not recognized`

**Error Message:**
```
error: cannot find attribute `hax` in this scope
  --> src/lib.rs:5:7
   |
5  | #[cfg(hax)]
   |       ^^^
```

**Cause:** The `hax` configuration flag is not set.

**Solutions:**
```bash
# Solution 1: Use cargo-hax command
cargo hax into fstar

# Solution 2: Set RUSTFLAGS manually
RUSTFLAGS="--cfg hax" cargo build

# Solution 3: Add to .cargo/config.toml
[build]
rustflags = ["--cfg", "hax"]
```

### Error: `hax_lib macros not found`

**Error Message:**
```
error: cannot find macro `requires` in this scope
  --> src/lib.rs:3:3
   |
3  | #[requires(x > 0)]
   |   ^^^^^^^^
```

**Cause:** Missing or incorrect `hax-lib` dependency.

**Solutions:**
```toml
# Cargo.toml
[dependencies]
hax-lib = { version = "*", features = ["macros"] }  # Enable macros feature
```

---

## Verification Failures

### Error: `Precondition might not hold`

**Error Message:**
```
error[E1001]: precondition might not hold
  --> src/lib.rs:15:5
   |
12 | #[requires(x < 100)]
   | -------------------- required precondition
...
15 |     unsafe_function(x);
   |     ^^^^^^^^^^^^^^^^^^ x might not be less than 100 here
```

**Cause:** Cannot prove that precondition is satisfied at call site.

**Solutions:**
```rust
// Solution 1: Add explicit check
fn caller(x: u32) {
    if x < 100 {
        unsafe_function(x);  // Now provably safe
    }
}

// Solution 2: Propagate requirement
#[requires(x < 100)]
fn caller(x: u32) {
    unsafe_function(x);  // Precondition inherited
}

// Solution 3: Add assumption (use carefully!)
fn caller(x: u32) {
    assume!(x < 100);  // Explicitly assume
    unsafe_function(x);
}
```

### Error: `Postcondition might not hold`

**Error Message:**
```
error[E1002]: postcondition might not hold
  --> src/lib.rs:8:1
   |
4  | #[ensures(|result| result > 0)]
   | -------------------------------- declared postcondition
...
8  | }
   | ^ function might return non-positive value
```

**Cause:** Function implementation doesn't guarantee postcondition.

**Solutions:**
```rust
// Solution 1: Fix implementation
#[ensures(|result| result > 0)]
fn get_positive() -> i32 {
    let x = compute();
    if x <= 0 {
        1  // Ensure positive result
    } else {
        x
    }
}

// Solution 2: Weaken postcondition
#[ensures(|result| result >= 0)]  // Weaker but provable
fn get_non_negative() -> i32 {
    compute().abs()
}

// Solution 3: Add intermediate assertions
#[ensures(|result| result > 0)]
fn get_positive() -> i32 {
    let x = compute();
    assert!(x > 0);  // Help the verifier
    x
}
```

### Error: `Loop invariant not maintained`

**Error Message:**
```
error[E1003]: loop invariant not maintained
  --> src/lib.rs:10:5
   |
7  | #[loop_invariant(sum <= MAX)]
   | ------------------------------ declared invariant
...
10 |     sum += arr[i];
   |     ^^^^^^^^^^^^^ invariant might be violated here
```

**Cause:** Loop body doesn't preserve the invariant.

**Solutions:**
```rust
// Solution 1: Strengthen the invariant
let mut sum = 0;
#[loop_invariant(sum <= i * MAX_ELEMENT && i <= arr.len())]
for i in 0..arr.len() {
    sum += arr[i];
}

// Solution 2: Add bounds checking
let mut sum = 0;
#[loop_invariant(sum <= MAX)]
for i in 0..arr.len() {
    if sum + arr[i] > MAX {
        break;  // Prevent overflow
    }
    sum += arr[i];
}

// Solution 3: Use checked arithmetic
let mut sum = 0u32;
for i in 0..arr.len() {
    sum = sum.saturating_add(arr[i]);
    assert!(sum <= u32::MAX);
}
```

### Error: `Assertion might fail`

**Error Message:**
```
error[E1004]: assertion might fail
  --> src/lib.rs:12:5
   |
12 |     assert!(x / y > 0);
   |     ^^^^^^^^^^^^^^^^^^ cannot prove assertion holds
```

**Cause:** Verifier cannot prove the assertion.

**Solutions:**
```rust
// Solution 1: Add preconditions
#[requires(y != 0 && x > 0)]
fn divide_positive(x: u32, y: u32) {
    assert!(x / y >= 0);  // Now provable
}

// Solution 2: Add intermediate steps
fn divide_check(x: u32, y: u32) {
    if y == 0 {
        return;
    }
    let result = x / y;
    if x > 0 && result > 0 {
        assert!(x / y > 0);  // More specific context
    }
}

// Solution 3: Use assumptions
fn divide_assume(x: u32, y: u32) {
    assume!(y > 0 && x > y);  // Strong assumption
    assert!(x / y > 0);        // Now holds
}
```

---

## Backend-Specific Errors

### F* Backend Errors

#### Error: `F* type checking failed`

**Error Message:**
```
error[E2001]: F* extraction failed
F* error: Subtyping check failed; expected type Prims.nat; got type Prims.int
```

**Cause:** Type mismatch in generated F* code.

**Solutions:**
```rust
// Solution 1: Use appropriate types
fn nat_function(x: u32) -> u32 {  // Use unsigned for nat
    x + 1
}

// Solution 2: Add refinements
#[ensures(|result| result >= 0)]
fn to_nat(x: i32) -> i32 {
    x.abs()  // Ensure non-negative
}

// Solution 3: Use F*-specific annotations
#[fstar::type("nat")]
fn explicit_nat(x: u32) -> u32 {
    x
}
```

#### Error: `Z3 resource limit exceeded`

**Error Message:**
```
error[E2002]: (Error) Unknown assertion failed
F* error: Z3 rlimit exceeded (resource count 100000)
```

**Cause:** SMT solver timeout on complex proof.

**Solutions:**
```rust
// Solution 1: Increase resource limit
#[fstar::options("--z3rlimit 1000000")]
fn complex_function() { /* ... */ }

// Solution 2: Split into smaller functions
fn part1() { /* ... */ }
fn part2() { /* ... */ }
fn complex_function() {
    part1();
    part2();
}

// Solution 3: Add SMT patterns
#[fstar::smt_pattern("(forall x. trigger_pattern x)")]
fn with_pattern() { /* ... */ }
```

### Lean Backend Errors

#### Error: `Panic-freedom proof failed`

**Error Message:**
```
error[E2010]: Lean extraction failed
Lean error: failed to prove panic freedom for arithmetic operation
```

**Cause:** Cannot prove absence of panics in arithmetic.

**Solutions:**
```rust
// Solution 1: Use checked arithmetic
fn safe_add(x: u32, y: u32) -> Option<u32> {
    x.checked_add(y)
}

// Solution 2: Add bounds preconditions
#[requires(x < u32::MAX / 2 && y < u32::MAX / 2)]
fn bounded_add(x: u32, y: u32) -> u32 {
    x + y  // Provably safe
}

// Solution 3: Use Lean-specific operators
fn lean_add(x: u32, y: u32) -> u32 {
    // Will generate x +? y in Lean
    x.wrapping_add(y)
}
```

### Coq Backend Errors

#### Error: `Inductive type extraction failed`

**Error Message:**
```
error[E2020]: Coq extraction failed
Coq error: Cannot handle nested inductives
```

**Cause:** Complex recursive types not supported.

**Solutions:**
```rust
// Solution 1: Flatten nested structure
// Instead of:
enum Tree<T> {
    Node(Box<Tree<T>>, T, Box<Tree<T>>)
}

// Use:
struct Tree<T> {
    left: Option<Box<Tree<T>>>,
    value: T,
    right: Option<Box<Tree<T>>>,
}

// Solution 2: Use indices instead of nesting
struct TreeNode<T> {
    value: T,
    left_idx: Option<usize>,
    right_idx: Option<usize>,
}
```

---

## Type System Errors

### Error: `Cannot unify types`

**Error Message:**
```
error[E3001]: type mismatch
  --> src/lib.rs:10:5
   |
10 |     let x: Int = y;
   |                  ^ expected `Int`, found `u32`
```

**Cause:** Type mismatch in assignment or function call.

**Solutions:**
```rust
// Solution 1: Use lift/abstraction
let x: Int = y.lift();

// Solution 2: Explicit conversion
let x: Int = Int::from(y);

// Solution 3: Change type annotation
let x: u32 = y;  // Keep concrete type
```

### Error: `Refinement type invariant violated`

**Error Message:**
```
error[E3002]: refinement invariant not satisfied
  --> src/lib.rs:15:5
   |
15 |     NonZero::new(0)
   |     ^^^^^^^^^^^^^^^ value 0 violates refinement predicate
```

**Cause:** Attempting to create refinement type with invalid value.

**Solutions:**
```rust
// Solution 1: Check before construction
fn make_nonzero(x: i32) -> Option<NonZero> {
    if x != 0 {
        Some(NonZero::new(x))
    } else {
        None
    }
}

// Solution 2: Use fallback value
fn make_nonzero_or_one(x: i32) -> NonZero {
    if x != 0 {
        NonZero::new(x)
    } else {
        NonZero::new(1)
    }
}

// Solution 3: Add precondition
#[requires(x != 0)]
fn make_nonzero_unchecked(x: i32) -> NonZero {
    NonZero::new(x)
}
```

---

## Macro Expansion Errors

### Error: `Macro expansion failed`

**Error Message:**
```
error[E4001]: macro expansion failed
  --> src/lib.rs:5:1
   |
5  | #[requires]
   | ^^^^^^^^^^^ expected expression argument
```

**Cause:** Incorrect macro syntax.

**Solutions:**
```rust
// Correct syntax examples:

// requires needs an expression
#[requires(x > 0)]

// ensures needs a closure
#[ensures(|result| result > 0)]

// loop_invariant needs a boolean expression
#[loop_invariant(i < len)]

// decreases needs a measure
#[decreases(n - i)]
```

### Error: `Attribute macro in wrong position`

**Error Message:**
```
error[E4002]: attribute can only be applied to functions
  --> src/lib.rs:10:1
   |
10 | #[requires(x > 0)]
   | ^^^^^^^^^^^^^^^^^^ cannot be applied to struct
```

**Cause:** Macro applied to wrong item type.

**Solutions:**
```rust
// Correct placements:

// Function contracts
#[requires(x > 0)]
fn function(x: u32) {}

// Loop specifications
#[loop_invariant(i < 10)]
for i in 0..10 {}

// Type refinements
#[refinement_type]
struct MyType {}
```

---

## Engine Errors

### Error: `Engine binary not found`

**Error Message:**
```
error[E5001]: hax-engine not found
The binary [hax-engine] was not found in your [PATH].
```

**Cause:** `hax` engine not installed or not in PATH.

**Solutions:**
```bash
# Solution 1: Install via setup script
./setup.sh

# Solution 2: Set engine path explicitly
export HAX_ENGINE_BINARY=/path/to/hax-engine

# Solution 3: Use OPAM
opam install hax-engine
eval $(opam env)

# Solution 4: Use Nix
nix develop
```

### Error: `AST extraction failed`

**Error Message:**
```
error[E5002]: failed to extract AST
Internal error: THIR extraction failed
```

**Cause:** Internal error in AST extraction.

**Solutions:**
```bash
# Solution 1: Clean and rebuild
cargo clean
cargo build

# Solution 2: Update rustc version
rustup update
rustup override set nightly-2024-01-01

# Solution 3: Simplify code
# Remove complex generics/macros temporarily

# Solution 4: Report bug with minimal example
cargo hax into fstar --verbose > debug.log
```

---

## Performance Issues

### Issue: Verification takes too long

**Symptoms:**
- Verification hangs or times out
- High CPU usage for extended periods
- Z3/SMT solver timeout errors

**Solutions:**

```rust
// Solution 1: Increase solver timeout
#[fstar::options("--z3rlimit 10000000")]

// Solution 2: Split complex functions
fn complex_function() {
    step1();
    step2();
    step3();
}

// Solution 3: Use lemmas
#[lemma]
fn helper_lemma() {
    // Prove once, reuse
}

// Solution 4: Weaken specifications
#[ensures(|result| result > 0)]  // Instead of complex formula
```

### Issue: Out of memory during verification

**Symptoms:**
- OOM errors
- Process killed
- System becomes unresponsive

**Solutions:**

```bash
# Solution 1: Increase memory limits
ulimit -v unlimited

# Solution 2: Use incremental verification
cargo hax into fstar --incremental

# Solution 3: Exclude non-critical parts
#[exclude]
fn non_critical_function() {}

# Solution 4: Use separate verification runs
cargo hax into fstar --include "module1::*"
cargo hax into fstar --include "module2::*"
```

---

## Environment and Setup Issues

### Error: `Incompatible Rust toolchain`

**Error Message:**
```
error[E6001]: rustc version mismatch
Expected: nightly-2024-01-01
Found: stable-1.75.0
```

**Solutions:**
```bash
# Install correct toolchain
rustup install nightly-2024-01-01
rustup override set nightly-2024-01-01

# Or let hax handle it
cargo hax into fstar  # Auto-installs correct version
```

### Error: `Missing dependencies`

**Error Message:**
```
error[E6002]: required dependency not found: FStar
```

**Solutions:**
```bash
# F* dependencies
opam install fstar
export FSTAR_HOME=$(opam var fstar:lib)

# HACL* dependencies
git clone https://github.com/hacl-star/hacl-star
export HACL_HOME=/path/to/hacl-star

# Complete setup
./setup.sh  # Installs all dependencies
```

---

## Debugging Strategies

### Strategy 1: Incremental Debugging

```rust
// Start with simple assertions
fn debug_step_by_step(x: u32) -> u32 {
    assert!(x < u32::MAX);  // Step 1
    let y = x + 1;
    assert!(y > x);         // Step 2
    y
}
```

### Strategy 2: Proof Logging

```rust
#[fstar::options("--log_queries")]
#[fstar::options("--debug Full")]
fn debug_with_logging() {
    // Check generated queries
}
```

### Strategy 3: Bisection

```rust
// Binary search for problematic code
#[cfg(not(debug_verification))]
fn complex_part() { /* ... */ }

#[cfg(debug_verification)]
fn complex_part() {
    // Simplified version
}
```

### Strategy 4: Ghost Code

```rust
#[ghost]
fn verification_helper(x: u32) -> bool {
    // Only for verification
    x < 100
}

fn main_function(x: u32) {
    assert!(verification_helper(x));
}
```

---

## Common Solutions

### Quick Fixes Checklist

1. **Update dependencies:**
   ```bash
   cargo update
   rustup update
   ```

2. **Clean build:**
   ```bash
   cargo clean
   rm -rf target/
   ```

3. **Check environment:**
   ```bash
   echo $HAX_ENGINE_BINARY
   echo $FSTAR_HOME
   echo $HACL_HOME
   ```

4. **Verbose output:**
   ```bash
   RUST_LOG=debug cargo hax into fstar --verbose
   ```

5. **Minimal example:**
   ```rust
   // Reduce to smallest failing case
   #[cfg(test)]
   mod test {
       #[test]
       fn minimal_repro() {
           // Minimal failing code
       }
   }
   ```

---

## Error Code Reference

### Quick Reference Table

| Code | Category | Common Cause | Quick Fix |
|------|----------|--------------|-----------|
| E0xxx | Compilation | Syntax/type errors | Fix Rust code |
| E1001 | Precondition | Missing requirement | Add checks/requires |
| E1002 | Postcondition | Weak implementation | Fix logic/ensures |
| E1003 | Invariant | Loop invariant | Strengthen invariant |
| E1004 | Assertion | Unprovable assert | Add context/assumes |
| E2001 | F* | Type mismatch | Fix types |
| E2002 | F* | Z3 timeout | Increase limits |
| E2010 | Lean | Panic possible | Use checked ops |
| E2020 | Coq | Complex types | Simplify structure |
| E3001 | Types | Type mismatch | Add conversions |
| E3002 | Refinement | Invalid value | Check invariants |
| E4001 | Macros | Bad syntax | Fix macro usage |
| E5001 | Engine | Not found | Install hax |
| E5002 | Engine | Extraction failed | Simplify code |
| E6001 | Environment | Wrong toolchain | Install correct |
| E6002 | Environment | Missing deps | Run setup.sh |

### Getting Help

When encountering persistent issues:

1. **Search issues:** https://github.com/hacspec/hax/issues
2. **Ask on Zulip:** https://hacspec.zulipchat.com/
3. **File bug report:** Include minimal reproduction example
