# Hax Examples and Usage Patterns

This document provides a comprehensive, hands-on guide to writing verifiable code with `hax`. It is structured as a series of practical examples and design patterns, starting from a "hello world" example and building up to complex, real-world cryptographic algorithms.

For a formal API-level reference, please see the [hax-lib API Documentation](hax-lib-api.md).

For a guide to proof strategies, see [Exhaustive Verification Techniques](verification-techniques.md).

## Table of Contents

### Part 1: Fundamentals
1. [Getting Started](#1-getting-started)
   - Minimal Example
   - Project Setup
   - Extracting to F*
2. [Basic Verification Patterns](#2-basic-verification-patterns)
   - Assertions vs. Assumptions
   - Function Contracts (`requires` / `ensures`)
   - Using Mathematical Integers (`Int`)

### Part 2: Core Patterns
3. [Mathematical Operations](#3-mathematical-operations)
   - Safe Division with Proofs
   - Power and Exponential Functions
4. [Loop Patterns](#4-loop-patterns)
   - Array Processing with Invariants
   - While Loops with Termination
5. [Refinement Types Examples](#5-refinement-types-examples)
   - Non-Zero Integer Type
   - Bounded Buffer Type

### Part 3: Advanced Applications
6. [Cryptographic Algorithms](#6-cryptographic-algorithms)
   - ChaCha20 Stream Cipher
   - Field Arithmetic (Barrett Reduction)
7. [Protocol Verification](#7-protocol-verification)
8. [Advanced Patterns](#8-advanced-patterns)
   - State Machines with Invariants
   - Recursive Data Structures (Termination)
9. [Real-World Examples](#9-real-world-examples)
   - SHA-256 Hash Function (Partial)
10. [Best Practices](#10-best-practices)

---

## Part 1: Fundamentals

## 1. Getting Started

This section covers the bare minimum setup to get a "hello world" verified project running with hax.

### Minimal Example

This is the simplest possible verifiable crate. It demonstrates a function with a contract (`#[requires]`, `#[ensures]`) and a function that uses an assumption (`assume!`) to prove safety.

```rust
// src/lib.rs
use hax_lib as hax;

/// A simple verified addition function.
/// The contract proves this is safe *only* for inputs < 100.
#[hax::requires(x < 100 && y < 100)]
#[hax::ensures(|result| result == x + y && result > x)]
pub fn verified_add(x: u32, y: u32) -> u32 {
    // We can add an assertion, which becomes a
    // proof obligation for the verifier.
    hax::assert!(x + y > x);
    x + y
}

/// A function that can panic, but we assume away the panic.
/// This is **UNSAFE** but demonstrates how to trust code.
pub fn divide_with_assumption(x: u32, y: u32) -> u32 {
    // `assume!` tells the verifier "you don't need to
    // prove this, just accept it as true."
    hax::assume!(y != 0);
    
    // Because of the assumption, `hax` can prove
    // the following division will not panic.
    x / y
}
```

### Project Setup

Your `Cargo.toml` must include `hax-lib` and enable the macros feature. You can also set project-wide backend options under `[package.metadata.hax]`.

```toml
# Cargo.toml
[package]
name = "my-verified-project"
version = "0.1.0"
edition = "2021"

[dependencies]
# The `macros` feature is essential. It pulls in `hax-lib-macros`
# which provides #[requires], #[ensures], #[loop_invariant], etc.
hax-lib = { version = "*", features = ["macros"] }

# Optional: Set global options for backends
[package.metadata.hax]
into.fstar.z3rlimit = 100
```

### Extracting to F*

The `cargo hax` command is your primary tool. It orchestrates the entire extraction and verification process.

```bash
# 1. Run the `cargo hax` subcommand to translate
#    your crate into F* files.
#    This reads your Rust code and generates `.fst` files.
cargo hax into fstar

# 2. (Optional) Pass options directly to the backend.
#    This overrides any settings in Cargo.toml.
cargo hax into fstar --z3rlimit 200 --output-dir proofs/

# 3. Verify the extracted code using the F* compiler.
#    This step actually runs the SMT solver (Z3)
#    on the generated .fst files to prove they are correct.
#    You must have F* and HACL* (for F* stdlib) in your environment.
cd proofs/fstar/extraction
fstar.exe --include $HACL_HOME/lib *.fst
```

**Common Workflow:** The most common development loop is:
1. Write/edit Rust code with `hax-lib` annotations.
2. Run `cargo hax into fstar`.
3. If it fails, read the error (e.g., "Postcondition might not hold").
4. Fix the Rust code (e.g., add an `assert!` or strengthen an invariant).
5. Repeat.

## 2. Basic Verification Patterns

These are the most common patterns you will use in everyday verification tasks.

### Assertions vs. Assumptions

This pattern demonstrates the critical difference between `assert!`, `assume!`, `assert_prop!`, and `debug_assert!`.

- `assert!(condition)`: Generates a **proof obligation**. The verifier _must_ prove this is true. This is your primary tool.
- `assume!(condition)`: Is **UNSAFE**. It tells the verifier to accept the condition as true without proof. It is necessary for trusting unverified code.
- `assert_prop!(proposition)`: Asserts a pure logical proposition (using `forall!`, `exists!`, `implies!`). This has no runtime equivalent and is only for the verifier.
- `debug_assert!(condition)`: Is **ignored by `hax`**. It has no effect on verification and is only for runtime debug builds.

```rust
use hax_lib as hax;

pub fn basic_assertions(x: u32, y: u32) {
    // 1. A PROVABLE ASSERTION
    // This is a runtime check (in debug) AND a proof obligation.
    // The verifier must prove `x < u32::MAX` is always true.
    hax::assert!(x < u32::MAX);

    // 2. AN UNSAFE ASSUMPTION
    // This has no runtime effect AND is NOT a proof obligation.
    // It's an "axiom" given to the verifier.
    // We use it here to "prove" the division is safe.
    hax::assume!(y > 0);
    let _ = x / y;

    // 3. A PURE LOGICAL PROPOSITION
    // This has no runtime effect but IS a proof obligation.
    // It uses quantifiers, which can't be run as Rust code.
    hax::assert_prop!(hax::forall(|z: u32| z >= 0));

    // 4. A DEBUG-ONLY RUNTIME CHECK
    // This is IGNORED by hax and generates NO proof obligation.
    // It only runs in debug builds.
    hax::debug_assert!(expensive_check(x, y), "Expensive check failed");
}

#[hax::exclude] // Exclude this helper from verification
fn expensive_check(_x: u32, _y: u32) -> bool { true }
```

### Function Contracts (`requires` / `ensures`)

This is the most fundamental pattern in `hax`. Contracts define a function's specification.

- `#[requires(...)]`: A **precondition**. This is a proof obligation for the _caller_ of the function. The function's body gets to assume it's true.
- `#[ensures(...)]`: A **postcondition**. This is a proof obligation for the _function's body_. The caller gets to assume it's true about the result.

```rust
use hax_lib as hax;

/// This function has a "strong" contract.
/// It *requires* the index to be in-bounds.
/// It *ensures* the returned value is the correct one.
#[hax::requires(index < array.len())]
#[hax::ensures(|result| result == array[index])]
pub fn safe_index(array: &[u32], index: usize) -> u32 {
    // We can prove this is safe because we can
    // assume the `#[requires]` contract holds.
    array[index]
}

/// This function has a "weaker" contract.
/// It has no preconditions, so it can be called with *any* index.
/// Its postcondition must account for the `None` case.
#[hax::ensures(|result|
    match result {
        Some(value) => index < array.len() && value == array[index],
        None => index >= array.len(),
    }
)]
pub fn checked_index(array: &[u32], index: usize) -> Option<u32> {
    // We must prove that this implementation
    // satisfies the `#[ensures]` contract for both
    // the Some and None cases.
    if index < array.len() {
        Some(array[index])
    } else {
        None
    }
}

// === Using the functions ===

fn use_contracts(my_array: &[u32]) {
    // To call `safe_index`, we MUST satisfy its precondition.
    if my_array.len() > 5 {
        // PROVABLE: `5 < my_array.len()` is true
        let val = safe_index(my_array, 5);
        hax::assert!(val == my_array[5]); // We can assume the `ensures`
    }
    
    // This call would FAIL verification because `hax`
    // cannot prove the precondition `10 < my_array.len()` holds.
    // let val = safe_index(my_array, 10);
    
    // `checked_index` has no preconditions,
    // so it's always safe to call.
    match checked_index(my_array, 10) {
        Some(val) => {
            // We can assume the `ensures` for the `Some` case
            hax::assert!(10 < my_array.len() && val == my_array[10]);
        }
        None => {
            // We can assume the `ensures` for the `None` case
            hax::assert!(10 >= my_array.len());
        }
    }
}
```

### Using Mathematical Integers (`Int`)

Rust's fixed-width types (like `u32`, `i64`) wrap around on overflow, which is a major source of bugs. `hax`'s `Int` type is an unbounded, mathematical integer used only in specifications.

This pattern shows how to prove your function is safe from overflow by "lifting" the concrete types into their mathematical counterparts and stating your contract using `Int`.

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt}; // Import the Int type and trait

/// This function uses `Int` to prove it is overflow-safe.
/// The `Int::from()` is an alias for `lift()`.
#[hax::requires(
    Int::from(x) + Int::from(y) <= Int::from(u32::MAX) &&
    Int::from(x) + Int::from(y) >= Int::from(u32::MIN)
)]
#[hax::ensures(|result| Int::from(result) == Int::from(x) + Int::from(y))]
pub fn safe_add(x: u32, y: u32) -> u32 {
    // The preconditions prove that this concrete `+`
    // will have the same result as the mathematical `+`
    // and will not overflow.
    x + y
}

/// This function demonstrates the `checked_add` pattern,
/// which is often simpler and more idiomatic.
#[hax::ensures(|result|
    match result {
        Some(sum) => Int::from(sum) == Int::from(x) + Int::from(y),
        None => Int::from(x) + Int::from(y) > Int::from(u32::MAX),
    }
)]
pub fn checked_add_spec(x: u32, y: u32) -> Option<u32> {
    x.checked_add(y)
}

/// This function shows how to use `int!` and `concretize`.
fn int_example() {
    // Use `int!` macro for mathematical literals
    let math_a = hax::int!(1_000_000_000_000);
    let math_b = hax::int!(2_000_000_000_000);
    let math_c = math_a + math_b;
    
    hax::assert!(math_c == hax::int!(3_000_000_000_000));
    
    // Use `lift()` (or `.to_int()`) to convert from machine types
    let a: u64 = 5;
    let math_a_u64 = a.to_int(); // or a.lift()
    hax::assert!(math_a_u64 == hax::int!(5));
    
    // Use `concretize()` to convert back.
    // This is a partial operation and can fail a proof
    // if the value is out of bounds.
    let concrete_c: u64 = math_c.concretize();
    
    // This proof would fail, as the value is too large:
    // let concrete_fail: u32 = math_c.concretize();
}
```

---

## Part 2: Core Patterns

## 3. Mathematical Operations

This section demonstrates patterns for verifying common mathematical algorithms, proving both correctness and safety (e.g., absence of panics).

### Safe Division with Proofs

Division by zero causes a panic. hax requires that every division is provably safe. You have two main patterns to achieve this:

- **Contract-based**: Add a `#[requires(divisor != 0)]` precondition. This pushes the burden of proof to the caller.
- **Type-based**: Return an `Option<T>` and prove your ensures contract matches the `checked_div` semantics.

```rust
use hax_lib as hax;
use hax_lib::int::Int;

/// PATTERN 1: Precondition-based safety
/// This function is fast (no runtime check) but can
/// only be called in a context where `divisor` is known to be non-zero.
#[hax::requires(divisor != 0)]
#[hax::ensures(|result|
    Int::from(result) * Int::from(divisor) + Int::from(dividend % divisor) == Int::from(dividend)
)]
pub fn safe_division(dividend: u32, divisor: u32) -> u32 {
    dividend / divisor
}

/// PATTERN 2: `Option`-based safety (Idiomatic Rust)
/// This function is total (can be called with any input)
/// and matches the semantics of `checked_div`.
#[hax::ensures(|result|
    match result {
        Some(value) => divisor != 0 &&
                       Int::from(value) * Int::from(divisor) + Int::from(dividend % divisor) == Int::from(dividend),
        None => divisor == 0,
    }
)]
pub fn checked_division(dividend: u32, divisor: u32) -> Option<u32> {
    dividend.checked_div(divisor)
}

/// PATTERN 3: Euclidean Division (Recursive)
/// This demonstrates proving correctness for a recursive algorithm.
/// The `#[decreases]` attribute is essential for proving termination.
#[hax::requires(divisor > 0)]
#[hax::ensures(|(q, r)|
    Int::from(dividend) == Int::from(divisor) * Int::from(q) + Int::from(r) &&
    Int::from(r) >= hax::int!(0) && Int::from(r) < Int::from(divisor)
)]
#[hax::decreases(dividend)]
pub fn euclidean_div_rec(dividend: u32, divisor: u32) -> (u32, u32) {
    if dividend < divisor {
        (0, dividend)
    } else {
        // `hax` proves termination by seeing that `dividend - divisor < dividend`
        let (q, r) = euclidean_div_rec(dividend - divisor, divisor);
        (q + 1, r)
    }
}
```

### Power and Exponential Functions

Proving properties of algorithms with internal loops or recursion requires invariants and termination measures.

- `#[decreases(exp)]`: Proves that the recursive power function terminates because exp gets smaller on every call.
- `#[loop_invariant(...)]`: Proves that the `fast_pow` iterative function is correct. The invariant `result * (b^exp) == initial_base^initial_exp` is a mathematical property that holds true at the start of every loop iteration, even as the variables `result`, `b`, and `exp` change.

```rust
use hax_lib as hax;
use hax_lib::int::{Int, ToInt};

/// PATTERN 1: Recursive power function
/// Proves correctness using mathematical induction and a decreases clause.
#[hax::decreases(exp)]
#[hax::ensures(|result|
    Int::from(result) == hax::int!(base).pow(exp.to_int())
)]
pub fn power(base: u64, exp: u32) -> u64 {
    if exp == 0 {
        1
    } else {
        // `hax` proves that if `power(base, exp - 1)` is correct
        // (the "inductive hypothesis"), then `base * ...` is also correct.
        base * power(base, exp - 1)
    }
}

/// PATTERN 2: Iterative fast exponentiation (exponentiation by squaring)
/// Proves correctness using a loop invariant.
#[hax::ensures(|result|
    Int::from(result) == hax::int!(base).pow(exp.to_int())
)]
pub fn fast_pow(base: u64, exp: u32) -> u64 {
    let mut result: u64 = 1;
    let mut b: u64 = base;
    let mut e: u32 = exp;
    
    // Ghost variables to store initial state for the invariant
    #[hax::ghost] let initial_base = base.to_int();
    #[hax::ghost] let initial_exp = exp.to_int();

    #[hax::loop_invariant(
        // This mathematical property is preserved by every iteration
        Int::from(result) * (Int::from(b).pow(e.to_int())) == initial_base.pow(initial_exp)
    )]
    #[hax::loop_decreases(e)] // Proves termination
    while e > 0 {
        if e % 2 == 1 {
            result = result * b;
        }
        b = b * b;
        e = e / 2;
    }

    // At the end, e == 0. The invariant becomes:
    // `result * (b^0) == initial_base^initial_exp`
    // `result * 1 == initial_base^initial_exp`
    // This proves our postcondition.
    result
}
```

## 4. Loop Patterns

Loops are where simple operations become complex. Loop invariants are the only way to prove properties about loops.

An invariant must be:
- True before the loop begins.
- Preserved by every iteration (i.e., if it's true at the start of an iteration, it's true at the end).
- Sufficient to prove the postcondition when the loop exits.

### Array Processing with Invariants

This pattern is for iterating over a collection to compute a result.

```rust
use hax_lib as hax;
use hax_lib::int::Int;

/// PATTERN 1: Summing an array (with overflow check)
/// This invariant proves that `sum` is the correct
/// sum of the subarray `arr[0..i]`.
#[hax::ensures(|result|
    match result {
        Some(sum) => Int::from(sum) == hax::forall(|i: usize|
                            implies(i < arr.len(), Int::from(arr[i]))
                       ).sum(), // This is pseudocode for `arr.iter().map(Int::from).sum()`
        None => true, // Could be strengthened
    }
)]
pub fn sum_array(arr: &[u32]) -> Option<u32> {
    let mut sum = 0u32;
    let mut i = 0;

    #[hax::loop_invariant(i <= arr.len())]
    #[hax::loop_invariant(
        // `sum` is the correct sum of the part we've processed
        Int::from(sum) == hax::forall(|j: usize|
                            implies(j < i, Int::from(arr[j]))
                       ).sum() // pseudocode
    )]
    #[hax::loop_decreases(arr.len() - i)]
    while i < arr.len() {
        match sum.checked_add(arr[i]) {
            Some(new_sum) => sum = new_sum,
            None => return None,
        }
        i += 1;
    }
    
    // After the loop, `i == arr.len()`.
    // The invariant implies: `sum == arr[0..arr.len()].iter().sum()`
    // This proves our postcondition.
    Some(sum)
}

/// PATTERN 2: Find maximum element
/// The invariant proves that `max` is the largest element
/// in the subarray `arr[0..i]` that has been processed so far.
#[hax::requires(arr.len() > 0)]
#[hax::ensures(|result|
    forall(|i: usize| implies(i < arr.len(), arr[i] <= result)) &&
    exists(|i: usize| implies(i < arr.len(), arr[i] == result))
)]
pub fn find_max(arr: &[u32]) -> u32 {
    let mut max = arr[0];
    let mut i = 1;

    #[hax::loop_invariant(i <= arr.len())]
    #[hax::loop_invariant(
        // `max` is the max of the subarray `arr[0..i]`
        forall(|j: usize| implies(j < i, arr[j] <= max))
    )]
    #[hax::loop_decreases(arr.len() - i)]
    while i < arr.len() {
        if arr[i] > max {
            max = arr[i];
        }
        i += 1;
    }
    
    // After the loop, `i == arr.len()`.
    // The invariant implies: `forall j < arr.len(), arr[j] <= max`
    // This proves our postcondition.
    max
}
```

### While Loops with Termination

This classic `binary_search` example demonstrates two crucial invariants:

- The `left <= right` invariant proves the bounds never cross.
- The `forall` invariants prove that the "search space" is correctly partitioned: everything to the left of `left` is smaller than the target, and everything to the right of `right` is greater or equal.
- `#[hax::loop_decreases(right - left)]` proves the loop terminates, because the search space `right - left` gets strictly smaller in every iteration.

```rust
use hax_lib as hax;

#[hax::ensures(|result|
    match result {
        Some(idx) => arr[idx] == target,
        None => forall(|i: usize| implies(i < arr.len(), arr[i] != target))
    }
)]
pub fn binary_search(arr: &[u32], target: u32) -> Option<usize> {
    let mut left = 0;
    let mut right = arr.len();

    #[hax::loop_invariant(left <= right && right <= arr.len())]
    #[hax::loop_invariant(
        // All items *before* `left` are smaller than target
        forall(|i: usize| implies(i < left, arr[i] < target))
    )]
    #[hax::loop_invariant(
        // All items *at or after* `right` are >= target
        forall(|i: usize| implies(i >= right && i < arr.len(), arr[i] >= target))
    )]
    #[hax::loop_decreases(right - left)]
    while left < right {
        let mid = left + (right - left) / 2;

        if arr[mid] == target {
            return Some(mid);
        } else if arr[mid] < target {
            // Target is in the right half.
            // We set `left = mid + 1`, which preserves the invariant.
            left = mid + 1;
        } else {
            // Target is in the left half.
            // We set `right = mid`, which preserves the invariant.
            right = mid;
        }
    }

    // Loop terminates when `left == right`.
    // If we're here, the item was not found.
    // The invariants prove that no item in the array equals target.
    None
}
```

## 5. Refinement Types Examples

A Refinement Type is a powerful pattern where you encode an invariant directly into the type system. This means the compiler and verifier can enforce properties for you, often with less manual effort.

You define a struct with `#[refinement_type]` and a field with `#[refinement]`, then implement the `Refinement` trait to define the invariant.

### Non-Zero Integer Type

This pattern creates a type `NonZeroI32` that cannot hold a zero. This is far safer than passing a regular `i32` and adding `#[requires(x != 0)]` to every function that uses it.

```rust
use hax_lib as hax;

/// 1. Define the struct with the refinement attributes.
#[hax::refinement_type]
pub struct NonZeroI32 {
    #[hax::refinement] // Mark the field that holds the value
    value: i32,
}

/// 2. Implement the `Refinement` trait to define the invariant.
impl hax::Refinement for NonZeroI32 {
    type InnerType = i32;

    /// This is the logical predicate (the "refinement").
    fn invariant(value: i32) -> hax::Prop {
        hax::Prop::from(value != 0)
    }
    
    // `new`, `get`, `get_ref` are auto-generated by the macro.
}

/// 3. (Optional) Create a "smart constructor" for safe creation.
impl NonZeroI32 {
    #[hax::ensures(|result|
        match result {
            Some(nz) => nz.value == value && value != 0,
            None => value == 0,
        }
    )]
    pub fn new_checked(value: i32) -> Option<Self> {
        // `into_checked` comes from the `RefineAs` trait,
        // which is also auto-implemented.
        value.into_checked()
    }
}

/// 4. Use the type to guarantee safety.
/// This function is *provably safe* from division by zero,
/// just from its type signature!
pub fn divide_by_nonzero(dividend: i32, divisor: NonZeroI32) -> i32 {
    // We can `deref` the `divisor` to get the inner `i32`.
    // The verifier *knows* `*divisor != 0` because of its type.
    dividend / *divisor
}
```

### Bounded Buffer Type

This pattern creates a `BoundedBuffer` that is guaranteed to never grow larger than `MAX_BUFFER_SIZE`. This is a powerful way to enforce resource limits and prevent denial-of-service attacks.

```rust
use hax_lib as hax;

const MAX_BUFFER_SIZE: usize = 1024;

/// 1. Define the struct.
#[hax::refinement_type]
pub struct BoundedBuffer {
    #[hax::refinement]
    data: Vec<u8>,
}

/// 2. Implement the invariant.
impl hax::Refinement for BoundedBuffer {
    type InnerType = Vec<u8>;
    fn invariant(data: Vec<u8>) -> hax::Prop {
        hax::Prop::from(data.len() <= MAX_BUFFER_SIZE)
    }
}

/// 3. Implement methods that use and preserve the invariant.
impl BoundedBuffer {
    /// The "unsafe" `new` from the trait is hidden.
    /// We provide a smart constructor that *checks* the invariant.
    #[hax::ensures(|result|
        match result {
            Some(buf) => buf.data.len() == data.len(),
            None => data.len() > MAX_BUFFER_SIZE,
        }
    )]
    pub fn new(data: Vec<u8>) -> Option<Self> {
        data.into_checked()
    }
    
    /// This method must *prove* it preserves the invariant.
    #[hax::requires(self.data.len() + other.data.len() <= MAX_BUFFER_SIZE)]
    #[hax::ensures(|_| self.data.len() <= MAX_BUFFER_SIZE)]
    pub fn append(&mut self, other: &BoundedBuffer) {
        // We get `other.data.len() <= MAX_BUFFER_SIZE` for free.
        // The `#[requires]` proves that `extend_from_slice`
        // will not violate *our* invariant.
        self.data.extend_from_slice(&other.data);
    }

    #[hax::ensures(|result| result.len() <= MAX_BUFFER_SIZE)]
    pub fn as_slice(&self) -> &[u8] {
        &self.data
    }
}
```

---

## Part 3: Advanced Applications

## 6. Cryptographic Algorithms

`hax` is purpose-built for verifying cryptographic code. These examples show how to combine mathematical reasoning, bit-level manipulation, and loop invariants to prove crypto primitives correct.

### ChaCha20 Stream Cipher

This pattern demonstrates verifying a real-world, high-performance cipher. The core logic is in `chacha20_quarter_round`, which involves complex bitwise operations (`^`, `rotate_left`) and `wrapping_add`. The `chacha20_rounds` function then proves that applying this round 10 times (20 rounds total) is correct, using a `#[loop_invariant]` to manage the state.

```rust
use hax_lib as hax;

type State = [u32; 16];
type Block = [u8; 64];
type ChaChaKey = [u8; 32];
type ChaChaIV = [u8; 12];

/// ChaCha20 quarter round.
/// This function is pure, stateless, and proven correct
/// by `hax`'s bit-vector arithmetic reasoning.
#[hax::transparent] // Mark as transparent to inline it
fn chacha20_quarter_round(
    a: usize, b: usize, c: usize, d: usize,
    mut state: State
) -> State {
    state[a] = state[a].wrapping_add(state[b]);
    state[d] ^= state[a];
    state[d] = state[d].rotate_left(16);

    state[c] = state[c].wrapping_add(state[d]);
    state[b] ^= state[c];
    state[b] = state[b].rotate_left(12);

    state[a] = state[a].wrapping_add(state[b]);
    state[d] ^= state[a];
    state[d] = state[d].rotate_left(8);

    state[c] = state[c].wrapping_add(state[d]);
    state[b] ^= state[c];
    state[b] = state[b].rotate_left(7);

    state
}

/// Main ChaCha20 rounds with loop invariant.
/// We prove that the state is correctly permuted after 10 double-rounds.
#[hax::ensures(|result| result == spec::chacha20_rounds(state))]
pub fn chacha20_rounds(state: State) -> State {
    let mut st = state;

    #[hax::loop_invariant(
        // The invariant states that after `_round` iterations,
        // `st` is equal to the spec function applied `_round` times.
        st == spec::chacha20_rounds_iter(state, _round)
    )]
    #[hax::loop_decreases(10 - _round)]
    for _round in 0..10 {
        // Column rounds
        st = chacha20_quarter_round(0, 4, 8, 12, st);
        st = chacha20_quarter_round(1, 5, 9, 13, st);
        st = chacha20_quarter_round(2, 6, 10, 14, st);
        st = chacha20_quarter_round(3, 7, 11, 15, st);

        // Diagonal rounds
        st = chacha20_quarter_round(0, 5, 10, 15, st);
        st = chacha20_quarter_round(1, 6, 11, 12, st);
        st = chacha20_quarter_round(2, 7, 8, 13, st);
        st = chacha20_quarter_round(3, 4, 9, 14, st);
    }
    st
}

#[hax::exclude] mod spec { /* ... spec functions ... */ }
```

### Field Arithmetic (Barrett Reduction)

This pattern demonstrates verifying a highly optimized mathematical algorithm against its simpler, formal specification. Barrett reduction is a fast way to compute `value % modulus`.

The `#[ensures]` contract is the specification:
- The result is in the correct range (`-MODULUS < result < MODULUS`).
- The result is mathematically equivalent (`result % MODULUS == value % MODULUS`).

This proof is complex and requires an SMT hint (`--z3rlimit`) and a `hax::fstar!` lemma call to help the solver.

```rust
use hax_lib as hax;
use hax_lib::int::Int;

pub type FieldElement = i32;

const BARRETT_SHIFT: i64 = 26;
const BARRETT_R: i64 = 0x4000000; // 2^26
const BARRETT_MULTIPLIER: i64 = 20159;
const FIELD_MODULUS: i32 = 3329;

/// Barrett reduction for Kyber field arithmetic.
#[hax::fstar::options("--z3rlimit 100")]
#[hax::requires(i64::from(value) >= -BARRETT_R && i64::from(value) <= BARRETT_R)]
#[hax::ensures(|result|
    let res_i = Int::from(result);
    let val_i = Int::from(value);
    let mod_i = Int::from(FIELD_MODULUS);
    // 1. Range check
    res_i > -mod_i && res_i < mod_i &&
    // 2. Equivalence check
    res_i.rem_euclid(mod_i) == val_i.rem_euclid(mod_i)
)]
pub fn barrett_reduce(value: FieldElement) -> FieldElement {
    // This optimized, non-obvious implementation...
    let t = i64::from(value) * BARRETT_MULTIPLIER;
    let t = t + (BARRETT_R >> 1);
    let quotient = (t >> BARRETT_SHIFT) as i32;
    let sub = quotient * FIELD_MODULUS;

    // ...is proven correct by hax.
    // We can even call an F* lemma to help the SMT solver.
    hax::fstar!(r"Math.Lemmas.cancel_mul_mod (v $quotient) 3329");

    value - sub
}
```

## 7. Protocol Verification

This pattern uses hax's ProVerif backend to formally verify security protocols. Instead of proving functional correctness, this proves abstract security properties like **secrecy** (an attacker can't learn the secret) and **authentication** (the client is really talking to the server).

The macros define the messages, identify constructors (`#[pv_constructor]`), and define the logic of the protocol participants (`#[process_init]`).

```rust
use hax_lib as hax;

/// 1. Define all messages in the protocol
#[hax::protocol_messages]
pub enum Message {
    ClientHello { nonce: [u8; 32] },
    ServerHello { nonce: [u8; 32], cert: Vec<u8> },
    ClientFinished { verification: [u8; 64] },
}

/// 2. Identify message constructors for ProVerif
#[hax::pv_constructor]
pub fn create_client_hello(nonce: [u8; 32]) -> Message {
    Message::ClientHello { nonce }
}

/// 3. Define the protocol logic for a participant
#[hax::process_init]
pub fn client_protocol(server_cert: Vec<u8>, secret_key: [u8; 32]) {
    // Generate random nonce (fresh value in ProVerif)
    let client_nonce = [0u8; 32]; // `fresh_nonce()`

    // Send client hello
    let hello = create_client_hello(client_nonce);
    send_message(hello); // `out(c, hello)` in ProVerif

    // Receive server hello
    // `in(c, msg)` in ProVerif
    if let Message::ServerHello { nonce: server_nonce, cert } = receive_message() {
        // Verify certificate
        hax::assume!(cert == server_cert);

        // Create verification data
        let verification = compute_verification(client_nonce, server_nonce, secret_key);

        // Send finished message
        let finished = Message::ClientFinished { verification };
        send_message(finished);
    }
}

// --- Helper functions (assumed or verified) ---
#[hax::exclude]
fn send_message(_msg: Message) { /* ... */ }
#[hax::exclude]
fn receive_message() -> Message { /* ... */ }
#[hax::exclude]
fn compute_verification(_c: [u8; 32], _s: [u8; 32], _k: [u8; 32]) -> [u8; 64] { /* ... */ }
```

## 8. Advanced Patterns

These patterns are for managing large, complex proofs.

### State Machines with Invariants

This pattern is for proving functional correctness of stateful systems. The core idea is to:

1. Define the state as a Rust enum.
2. Define the machine as a struct holding the state.
3. Implement transitions as methods with `#[requires]` and `#[ensures]` clauses that enforce the state transition logic.

This formally proves that, e.g., you can only call `step()` when the state is `Running`, and that `start()` always moves the state from `Init` to `Running`.

```rust
use hax_lib as hax;

#[derive(Clone, Copy, PartialEq)]
enum State {
    Init,
    Running,
    Stopped,
}

pub struct StateMachine {
    state: State,
    counter: u32,
}

impl StateMachine {
    /// The constructor establishes the initial state invariant.
    #[hax::ensures(|result| result.state == State::Init && result.counter == 0)]
    pub fn new() -> Self {
        StateMachine {
            state: State::Init,
            counter: 0,
        }
    }

    /// `start` transition: Init -> Running
    #[hax::requires(self.state == State::Init)]
    #[hax::ensures(|_| self.state == State::Running)]
    pub fn start(&mut self) {
        self.state = State::Running;
    }

    /// `step` transition: Running -> Running
    #[hax::requires(self.state == State::Running)]
    #[hax::ensures(|_| self.state == State::Running)]
    #[hax::ensures(|_| self.counter == hax::old(self.counter) + 1)]
    pub fn step(&mut self) {
        // `hax::old(...)` captures the value at the start of the function
        self.counter += 1;
    }

    /// `stop` transition: Running -> Stopped
    #[hax::requires(self.state == State::Running)]
    #[hax::ensures(|_| self.state == State::Stopped)]
    pub fn stop(&mut self) {
        self.state = State::Stopped;
    }
}
```

### Recursive Data Structures (Termination)

When verifying recursive data structures like trees or linked lists, the primary challenge is proving termination on recursive functions. You do this with the `#[decreases(self)]` attribute, which tells hax to use structural recursion as the termination measure (i.e., the left and right subtrees are structurally smaller than `self`).

```rust
use hax_lib as hax;

pub enum Tree<T> {
    Leaf(T),
    Node {
        value: T,
        left: Box<Tree<T>>,
        right: Box<Tree<T>>,
    },
}

impl Tree<u32> {
    /// Proves termination using the structure of `self`.
    #[hax::decreases(self)]
    pub fn sum(&self) -> u32 {
        match self {
            Tree::Leaf(v) => *v,
            Tree::Node { value, left, right } => {
                // `hax` proves that `left` and `right` are
                // structurally smaller than `self`.
                value + left.sum() + right.sum()
            }
        }
    }

    /// Proves a postcondition (`result >= 1`)
    /// using structural recursion.
    #[hax::ensures(|result| result >= 1)]
    #[hax::decreases(self)]
    pub fn size(&self) -> usize {
        match self {
            Tree::Leaf(_) => 1,
            Tree::Node { left, right, .. } => {
                1 + left.size() + right.size()
            }
        }
    }
}
```

## 9. Real-World Examples

This section shows how all the previous patterns are combined to verify a complete, real-world algorithm like SHA-256.

### SHA-256 Hash Function (Partial)

This example combines:
- **Array Processing**: `state` and `block` manipulation.
- **Bitwise Arithmetic**: `ch`, `maj`, `sigma0`, `sigma1` functions.
- **Loop Invariants**: Two complex for loops (message schedule and compression) must be proven correct.
- **Mathematical Correctness**: The final state update must be proven to match the SHA-256 specification.

```rust
use hax_lib as hax;

const K: [u32; 64] = [
    0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5,
    0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
    // ... rest of constants ...
];

// Helper functions (verified to be pure bitwise ops)
#[hax::transparent]
fn ch(x: u32, y: u32, z: u32) -> u32 { (x & y) ^ (!x & z) }
#[hax::transparent]
fn maj(x: u32, y: u32, z: u32) -> u32 { (x & y) ^ (x & z) ^ (y & z) }
#[hax::transparent]
fn sigma0(x: u32) -> u32 { x.rotate_right(2) ^ x.rotate_right(13) ^ x.rotate_right(22) }
#[hax::transparent]
fn sigma1(x: u32) -> u32 { x.rotate_right(6) ^ x.rotate_right(11) ^ x.rotate_right(25) }

#[hax::ensures(|_| state == spec::sha256_compress(hax::old(state), block))]
pub fn sha256_compress(state: &mut [u32; 8], block: &[u8; 64]) {
    let mut w = [0u32; 64];

    // 1. Message schedule loop
    #[hax::loop_invariant(i <= 16)]
    for i in 0..16 { 
        // Copy block into w...
        w[i] = u32::from_be_bytes([
            block[i * 4],
            block[i * 4 + 1],
            block[i * 4 + 2],
            block[i * 4 + 3],
        ]);
    }

    #[hax::loop_invariant(i >= 16 && i <= 64)]
    #[hax::loop_invariant(
        forall(|j: usize| implies(j < i, w[j] == spec::message_schedule(block, j)))
    )]
    for i in 16..64 {
        let s0 = w[i-15].rotate_right(7) ^ w[i-15].rotate_right(18) ^ (w[i-15] >> 3);
        let s1 = w[i-2].rotate_right(17) ^ w[i-2].rotate_right(19) ^ (w[i-2] >> 10);
        w[i] = w[i-16].wrapping_add(s0).wrapping_add(w[i-7]).wrapping_add(s1);
    }

    // 2. Compression loop
    let mut a = state[0]; let mut b = state[1]; let mut c = state[2];
    let mut d = state[3]; let mut e = state[4]; let mut f = state[5];
    let mut g = state[6]; let mut h = state[7];

    #[hax::loop_invariant(
        // Invariant relates a-h to the spec's state at round i
        spec::compression_vars(a,b,c,d,e,f,g,h) == spec::compression_state(state, w, i)
    )]
    for i in 0..64 {
        let t1 = h.wrapping_add(sigma1(e)).wrapping_add(ch(e, f, g))
                  .wrapping_add(K[i]).wrapping_add(w[i]);
        let t2 = sigma0(a).wrapping_add(maj(a, b, c));
        h = g; g = f; f = e; e = d.wrapping_add(t1);
        d = c; c = b; b = a; a = t1.wrapping_add(t2);
    }

    // 3. Final state update
    state[0] = state[0].wrapping_add(a);
    state[1] = state[1].wrapping_add(b);
    state[2] = state[2].wrapping_add(c);
    state[3] = state[3].wrapping_add(d);
    state[4] = state[4].wrapping_add(e);
    state[5] = state[5].wrapping_add(f);
    state[6] = state[6].wrapping_add(g);
    state[7] = state[7].wrapping_add(h);
}

#[hax::exclude] 
mod spec { 
    // Specification functions for verification
    pub fn sha256_compress(_state: &[u32; 8], _block: &[u8; 64]) -> [u32; 8] {
        unimplemented!("Spec function")
    }
    pub fn message_schedule(_block: &[u8; 64], _i: usize) -> u32 {
        unimplemented!("Spec function")
    }
    pub fn compression_vars(_a: u32, _b: u32, _c: u32, _d: u32, _e: u32, _f: u32, _g: u32, _h: u32) -> [u32; 8] {
        unimplemented!("Spec function")
    }
    pub fn compression_state(_state: &[u32; 8], _w: [u32; 64], _i: usize) -> [u32; 8] {
        unimplemented!("Spec function")
    }
}
```

## 10. Best Practices

A summary of best practices for writing high-quality, verifiable code with hax.

1. **Start Simple, Then Add Contracts**: Begin by writing standard Rust code. Get it to pass tests. Then, add simple `assert!` statements. Finally, graduate to full `#[requires]` / `#[ensures]` contracts.

2. **Use Mathematical Integers (`Int`) in Specifications**: Always use `Int` in contracts (`#[requires]`, `#[ensures]`, `#[loop_invariant]`) when reasoning about arithmetic. This avoids all overflow/wrapping complexity.

3. **Handle Panics Explicitly**: hax proves panic-freedom. Every division, index, and arithmetic operation must be proven safe. Use `#[requires(y != 0)]` for division, `#[requires(i < len)]` for indexing, and `checked_add` or `Int`-based contracts for arithmetic.

4. **Use Refinement Types for Common Invariants**: Don't write `#[requires(x != 0)]` on ten different functions. Create a `NonZeroI32` refinement type once and use it in all ten function signatures.

5. **Separate Specification from Implementation**: For complex algorithms, create a simple, obviously correct "spec" function (marked `#[hax::exclude]`). Then, prove that your optimized implementation `#[ensures(|result| result == spec_function(...))]`.

6. **Use `#[opaque]` for Abstraction**: Don't re-verify the same function 100 times. Verify it once, then mark it `#[opaque]`. Callers will now trust its contract, making verification modular and fast.

7. **Use Lemmas for Reusable Proofs**: If you find yourself writing the same `assert_prop!(...)` in multiple places, move it into a `#[lemma]` function. Call the lemma to import the proof.

8. **Document Invariants Clearly**: Write down your invariants (for loops or data structures) in plain English comments before you try to write them as hax code.

9. **Write Tests for Your Specifications**: A specification can be "correct" but impossible to satisfy (e.g., `#[ensures(|result| result > 0 && result < 0)]`). Your regular Rust tests (`#[test]`) are your first line of defense against contradictory or nonsensical specs.

10. **Leverage Backend-Specific Features**: Use `#[hax::fstar::options("--z3rlimit 100")]` to give the SMT solver more time, or `hax::fstar!(r"...")` to call F* lemmas directly when automatic verification struggles.

---

## "Conclusion"

This guide has covered the complete spectrum of verification with hax, from simple assertions to complex cryptographic protocols. The key to success is:

- **Start incrementally**: Add verification to small pieces of code first.
- **Think mathematically**: Use `Int` and logical propositions liberally.
- **Write clear specifications**: Your contracts are documentation for both humans and machines.
- **Leverage the type system**: Refinement types encode invariants once and reuse them everywhere.

For more information, consult the [hax-lib API Documentation](hax-lib-api.md) and [Exhaustive Verification Techniques](verification-techniques.md).
