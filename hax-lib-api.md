# hax-lib API Reference Documentation

This document provides an exhaustive API reference for the hax-lib crate. hax-lib is the core library that you use as a dependency in your Rust code to enable verification with hax. It provides the fundamental types, traits, macros, and attributes for writing formal specifications.

## Table of Contents

1.  [Core Modules](#core-modules)
2.  [Mathematical Types](#mathematical-types)
3.  [Logical Propositions](#logical-propositions)
4.  [Assertions and Assumptions](#assertions-and-assumptions)
    -   [`assert!`](#assert---generate-proof-obligations)
    -   [`assume!`](#assume---introduce-assumptions)
    -   [`assert_prop!`](#assert_prop---assert-logical-propositions)
    -   [`debug_assert!`](#debug_assert---debug-only-assertions)
5.  [Abstraction System](#abstraction-system)
    -   [Abstraction Trait](#abstraction-trait)
    -   [Concretization Trait](#concretization-trait)
6.  [Refinement Types](#refinement-types)
    -   [Refinement Trait](#refinement-trait)
    -   [RefineAs Trait](#refineas-trait)
    -   [`#[refinement_type]` Macro](#refinement_type-macro-usage)
7.  [Function Contracts](#function-contracts)
    -   [`#[requires(...)]`](#requires---preconditions)
    -   [`#[ensures(|result| ...)]`](#ensures---postconditions)
    -   [`#[lemma]`](#lemma---proof-only-functions)
8.  [Loop Specifications](#loop-specifications)
    -   [`#[loop_invariant(...)]`](#loop_invariant---loop-invariants)
    -   [`#[decreases(...)]`](#decreases---function-termination)
    -   [`#[loop_decreases(...)]`](#loop_decreases---loop-termination)
9.  [Visibility and Abstraction Macros](#visibility-and-abstraction-macros)
    -   [`#[include]` / `#[exclude]`](#include--exclude)
    -   [`#[opaque]` / `#[opaque_type]`](#opaque--opaque_type)
    -   [`#[transparent]`](#transparent)
10. [Backend-Specific APIs](#backend-specific-apis)
    -   [F* Backend](#f-backend)
    -   [Protocol Verification (ProVerif)](#protocol-verification-proverif)
11. [Internal Markers (Reference)](#internal-markers-reference)

## Core Modules

### Module Structure
`hax-lib` uses a different source file depending on the compilation target.
- When compiled with `cargo hax` (`cfg(hax)`): `src/implementation.rs` is used. This file contains the "real" implementations of types like `Int` and `Prop`, which are designed to be extracted by the `hax-engine`.
- When compiled with `cargo build` (`cfg(not(hax))`): `src/dummy.rs` is used. This file provides minimal, no-op, or panicking stubs for the verification-specific types and functions, allowing your Rust code to compile normally without the hax toolchain.

```
hax_lib/
├── src/
│   ├── lib.rs              // Main entry point, re-exports
│   ├── implementation.rs   // (cfg(hax)) Real implementations
│   ├── dummy.rs            // (cfg(not(hax))) Stub implementations
│   ├── abstraction.rs    // Abstraction traits
│   ├── prop.rs           // Logical propositions
│   ├── int/
│   │   └── mod.rs        // Mathematical integers
│   └── proc_macros.rs    // Re-exports from hax-lib-macros
```

## Feature Flags
```
#[features]
default = ["macros"]
# This feature enables all the procedural attribute macros
# (like #[requires], #[ensures], etc.) by depending on
# the `hax-lib-macros` crate.
macros = ["hax-lib-macros"]
```

## Mathematical Types

### `Int` - Mathematical Integers
Provides an unbounded, mathematical integer type (`Int`) for use in specifications. This type is crucial for reasoning about arithmetic without concerns for machine-level overflow or wrapping.

### Type Definition
```rust
// Internally wraps `num_bigint::BigInt`
#[derive(Copy, Clone, Eq, PartialEq, Ord, PartialOrd, Debug)]
pub struct Int(BigInt);
```

### Constructors and Literals
The primary way to create `Int` literals is with the `int!` macro.

```rust
#[cfg(feature = "macros")]
pub use hax_lib_macros::int;

// Usage:
let x: Int = int!(42);
let y: Int = int!(-1000000000000000000000000);
let z: Int = int!(0xDEADBEEF);
```

### Arithmetic Operations
`Int` supports all standard arithmetic operations.

```rust
impl Add for Int {
    type Output = Self;
    fn add(self, other: Self) -> Self::Output;
}

impl Sub for Int {
    type Output = Self;
    fn sub(self, other: Self) -> Self::Output;
}

impl Mul for Int {
    type Output = Self;
    fn mul(self, other: Self) -> Self::Output;
}

impl Div for Int {
    type Output = Self;
    fn div(self, other: Self) -> Self::Output;
}

impl Neg for Int {
    type Output = Self;
    fn neg(self) -> Self::Output;
}
Special Operationsimpl Int {
    /// Calculate 2^self.
    /// Panics if self is negative or doesn't fit in a u32.
    pub fn pow2(self) -> Self;

    /// Euclidean remainder (always non-negative).
    pub fn rem_euclid(&self, v: Self) -> Self;
}
```
### Conversion (Abstraction & Concretization)
`Int` is integrated with the abstraction system.
```rust
// Abstraction (Lifting): Machine integer -> Int
// This is infallible.
impl Abstraction for u32 {
    type AbstractType = Int;
    fn lift(self) -> Int;
}
// ... and for u8, u16, u64, u128, usize
// ... and for i8, i16, i32, i64, i128, isize

// Concretization: Int -> Machine integer
// This is a partial operation and can panic if the Int
// is out of bounds for the target machine type.
impl Concretization<u32> for Int {
    fn concretize(self) -> u32;
}
// ... and for all other machine integer types

// Trait for ergonomic lifting
pub trait ToInt {
    fn to_int(self) -> Int;
}

impl<T: Abstraction<AbstractType = Int>> ToInt for T {
    fn to_int(self) -> Int { self.lift() }
}
```

## Logical Propositions

### `Prop` - Logical Propositions
A type representing a logical proposition. This is distinct from bool. A `Prop` may not be computable at runtime (e.g., if it contains a `forall!`).

### Type Definition
```rust
// The runtime representation is a bool, but its
// meaning in `hax` is as a logical formula.
#[derive(Clone, Copy, Debug)]
pub struct Prop(bool);
Constructors and OperationsThe primary way to build Prop values is via conversion from bool, logical macros, and methods.impl Prop {
    pub const fn from_bool(b: bool) -> Self;
    pub fn and(self, other: impl Into<Self>) -> Self;
    pub fn or(self, other: impl Into<Self>) -> Self;
    pub fn not(self) -> Self;
    pub fn implies(self, other: impl Into<Self>) -> Self;
    pub fn eq(self, other: impl Into<Self>) -> Self;
    pub fn ne(self, other: impl Into<Self>) -> Self;
}

// Conversion from bool is the most common way
impl From<bool> for Prop {
    fn from(value: bool) -> Self;
}

// Ergonomic operators
impl<T: Into<Prop>> BitAnd<T> for Prop { // &
    type Output = Prop;
    fn bitand(self, rhs: T) -> Self::Output;
}

impl<T: Into<Prop>> BitOr<T> for Prop { // |
    type Output = Prop;
    fn bitor(self, rhs: T) -> Self::Output;
}

impl Not for Prop { // !
    type Output = Prop;
    fn not(self) -> Self::Output;
}

// Lifting a bool to a Prop
impl Abstraction for bool {
    type AbstractType = Prop;
    fn lift(self) -> Prop;
}
```

### Quantifier Macros]
These macros are the only way to introduce quantifiers into propositions.

#### `forall!`

Universal quantification: `∀(x: T), phi(x)`. 

```rust
/// Universal quantification: `∀(x: T), phi(x)`

pub fn forall<T, U: Into<Prop>>(f: impl Fn(T) -> U) -> Prop;

// Example
assert_prop!(forall(|x: u32| x >= 0));
exists!Existential quantification: ∃(x: T), phi(x)/// Existential quantification: ∃(x: T), phi(x)
pub fn exists<T, U: Into<Prop>>(f: impl Fn(T) -> U) -> Prop;

// Example
assert_prop!(exists(|x: i32| x < 0));
implies!Logical implication: a ==> b/// Logical implication: a ==> b
pub fn implies(lhs: impl Into<Prop>, rhs: impl Into<Prop>) -> Prop;

// Example
assert_prop!(implies(x > 0, x * 2 > x));
```

## Assertions and Assumptions

These are the core macros for inserting proof logic inside a function body.
    
For a detailed guide, see: [Hax Macro System Documentation](macro-system-complete.md). 

### `assert!` - Generate Proof Obligations

Generates a proof obligation that must be proven by the backend verifier. In standard Rust, this expands to ::core::assert!.
```rust
#[macro_export]
macro_rules! assert { /* ... */ }

// When compiled with hax, calls:
#[cfg(hax)]
pub fn assert(_formula: bool) {}

// Example
#[requires(x < 100)]
fn my_fn(x: u32) {
    assert!(x < 100); // This is provable from the precondition
}
```

### `assume!` - Introduce Assumptions
Introduces a fact as true without proof. This is UNSAFE and can lead to unsound proofs. In standard Rust, this expands to nothing (()).
```rust
#[macro_export]
macro_rules! assume { /* ... */ }

// When compiled with hax, calls:
#[cfg(hax)]
pub fn assume(_formula: Prop) {}

// Example
// Trust that an external, unverified function is correct
let y = external_lib::complex_op(x);
assume!(y == x * 2);
assert!(y > x); // This is now provable
```

### `assert_prop!` - Assert Logical Propositions
Asserts a pure logical proposition (of type Prop). This is required when using quantifiers. In standard Rust, this expands to nothing.
```rust
#[macro_export]
macro_rules! assert_prop { /* ... */ }

// When compiled with hax, calls:
#[cfg(hax)]
pub fn assert_prop(_formula: Prop) {}

// Example
fn find_max(arr: &[u32]) -> u32 {
    let max_val = /* ... */;
    // Prove that the returned value is >= all elements
    assert_prop!(forall(|i: usize|
        implies(i < arr.len(), arr[i] <= max_val)
    ));
    max_val
}
```

### `debug_assert!` - Debug-Only Assertions
Completely ignored by hax (expands to nothing). In standard Rust, this expands to ::core::debug_assert!.
```rust
#[macro_export]
macro_rules! debug_assert { /* ... */ }

// Example
// Use for expensive runtime checks that you do
// not want to include in the formal proof.
debug_assert!(is_sorted(my_vec), "Vector must be sorted!");
```

## Abstraction System
`hax` uses a trait-based system to "lift" concrete Rust types (like `u32`) into their abstract mathematical counterparts (like `Int`).

### `Abstraction` Trait
Maps concrete values to idealized abstract versions.

```rust
pub trait Abstraction {
    /// The ideal type values should be mapped to
    type AbstractType;

    /// Maps a concrete value to its abstract counterpart
    fn lift(self) -> Self::AbstractType;
}

// Example
let x: u32 = 10;
let y: Int = x.lift(); // y is Int(10)
```

### `Concretization` Trait
Maps abstract values back to concrete ones. This operation can fail (panic) if the abstract value is not representable.
```rust
pub trait Concretization<T> {
    /// Maps an abstract value to its concrete counterpart
    fn concretize(self) -> T;
}

// Example
let x: Int = int!(10);
let y: u32 = x.concretize(); // y is 10

let z: Int = int!(1_000_000_000_000);
let w: u32 = z.concretize(); // This will panic!
```

## Refinement Types
Refinement types combine a base type (like `i32`) with a logical invariant (like `value > 0`).

For a detailed guide, see: [Hax Macro System Documentation](macro-system-complete.md).

### Refinement Trait
The core trait for defining a refinement type. This is usually implemented automatically by the `#[refinement_type]` macro.

```rust
pub trait Refinement {
    /// The base type being refined
    type InnerType;

    /// Smart constructor capturing an invariant.
    /// In `hax`, this generates a proof obligation:
    /// the caller must prove `invariant(x)` holds.
    fn new(x: Self::InnerType) -> Self;

    /// Destructor for the refined type
    fn get(self) -> Self::InnerType;

    /// Gets a reference to the inner value
    fn get_ref(&self) -> &Self::InnerType;

    /// The logical predicate that defines the refinement.
    fn invariant(value: Self::InnerType) -> Prop;
}
```

### `RefineAs` Trait
Provides the `into_checked` method for safely constructing a refinement type from a base type.

```rust
pub trait RefineAs<RefinedType: Refinement> {
    /// Smart constructor that checks the invariant.
    /// In `hax`, generates a proof obligation.
    /// In standard Rust, checks at runtime in debug mode.
    fn into_checked(self) -> Option<RefinedType>;
}
```

### `#[refinement_type]` Macro Usage
```rust
use hax_lib::*;

// 1. Define the struct
#[refinement_type]
pub struct PositiveInt {
    #[refinement] // Mark the field
    value: i32,
}

// 2. Implement the invariant
impl Refinement for PositiveInt {
    type InnerType = i32;
    fn invariant(value: i32) -> Prop {
        Prop::from(value > 0)
    }
    // `new`, `get`, etc. are auto-generated
    //  by the `#[refinement_type]` macro.
}

// 3. Use it
fn get_positive(x: i32) -> Option<PositiveInt> {
    x.into_checked() // Uses RefineAs::into_checked
}

fn needs_positive(p: PositiveInt) {
    // We can assume p.get_ref() > 0
    assert!(*p.get_ref() > 0);
}
```

## Function Contracts

These are procedural macros that attach specifications to function signatures.

For a detailed guide, see: [Hax Macro System Documentation](macro-system-complete.md).

### `#[requires(...)]` - Preconditions

Specifies a condition that must be proven by the caller of the function. The function body can then assume this condition is true.
```rust
#[hax_lib::requires(y != 0)]
fn safe_divide(x: u32, y: u32) -> u32 {
    x / y // This is proven safe
}
```

### `#[ensures(|result| ...)]` - Postconditions
Specifies a condition that must be proven by the function's body. The caller can then assume this condition holds for the returned value.
```rust
#[hax_lib::requires(x < u32::MAX)]
#[hax_lib::ensures(|result| result > x)]
fn increment(x: u32) -> u32 {
    x + 1 // We must prove x + 1 > x
}
```

### `#[lemma]` - Proof-Only Functions
Defines a "ghost" function that exists only for verification. It is erased during normal compilation. Lemmas are used to prove and reuse mathematical properties.
```rust
#[hax_lib::lemma]
#[hax_lib::ensures(x + y == y + x)]
fn addition_is_commutative(x: Int, y: Int) {
    // The proof (often just `assert_prop!`)
    assert_prop!(x + y == y + x);
}

fn my_func(a: Int, b: Int) {
    addition_is_commutative(a, b); // Call the lemma
    // The verifier now knows `a + b == b + a`
    assert!(a + b == b + a);
}
```
## Loop Specifications

These macros are used to prove properties of loops (which represent unbounded computation).

For a detailed guide, see: [Hax Macro System Documentation](macro-system-complete.md).

### `#[loop_invariant(...)]` - Loop Invariants

Specifies a property that is true before the loop, and is preserved by every single iteration.
```rust
let mut sum = 0u32;
let mut i = 0;
#[loop_invariant(i <= arr.len())]
#[loop_invariant(sum == arr[..i].iter().sum())]
while i < arr.len() {
    sum += arr[i];
    i += 1;
}
assert!(sum == arr.iter().sum()); // Provable from invariant
```

### `#[decreases(...)]` - Function Termination
Specifies a "measure" (a value that must decrease) for a recursive function to prove that it terminates.
```rust
#[hax_lib::decreases(n)]
fn factorial(n: u32) -> u64 {
    if n == 0 { 1 } else { n * factorial(n - 1) }
}
```

### `#[loop_decreases(...)]` - Loop Termination
Specifies a "measure" for a loop to prove that it terminates.
```rust
let mut i = n;
#[hax_lib::loop_decreases(i)]
while i > 0 {
    i -= 1;
}
```

## Visibility and Abstraction Macros
These macros control what the hax extractor sees, allowing you to manage proof complexity.

For a detailed guide, see: [Hax Macro System Documentation](macro-system-complete.md).

### `#[include]` / `#[exclude]`

Control which items are (or are not) translated.
```rust
#[hax_lib::exclude] // Skip this debug function
fn debug_print_state(s: &State) { /* ... */ }

#[hax_lib::include] // Force-include this lemma
#[hax_lib::lemma]
fn my_helper_lemma() { /* ... */ }
```

### `#[opaque]` / `#[opaque_type]`
Hide the _implementation_ of a function or type from the verifier. Callers can only see its signature and contracts. This is the primary tool for abstraction and modular proofs.
```rust
#[hax_lib::opaque]
#[hax_lib::requires(key.len() == 32)]
#[hax_lib::ensures(|result| result.len() == 16)]
fn aes_encrypt(key: &[u8], block: &[u8]) -> Vec<u8> {
    // ... complex, optimized, unverified code ...
}

fn use_aes(key: &[u8], block: &[u8]) {
    assume!(key.len() == 32);
    let ciphertext = aes_encrypt(key, block);
    // Verifier assumes `ensures` holds:
    assert!(ciphertext.len() == 16);
}
```

### `#[transparent]`
Forces the extractor to inline a function's body. This is the opposite of `#[opaque]`.
```
#[hax_lib::transparent]
fn simple_helper(x: u32) -> u32 { x + 1 }
```

## Backend-Specific APIs
These are "escape hatches" for interacting directly with a specific verification backend. They are not portable.

### F* Backend

#### `fstar!` Macro - Inline F* Code
Injects a raw string of F* code into the extracted file. $-prefixed variables are substituted.
```rust
hax_lib::fstar!(r#"
    (* Call an F* library lemma *)
    Math.Lemmas.lemma_div_mod (v $x) (v $y)
"#);
```

#### F* Options

Set command-line flags for the F* solver for a specific function.
```rust
#[hax_lib::fstar::options("--z3rlimit 100 --fuel 2")]
fn very_complex_proof() { /* ... */ }
```

### Protocol Verification (ProVerif)
A set of macros for verifying security protocols.
```rust
#[hax_lib::protocol_messages]
enum Messages {
    Request { id: u32 },
    Response { id: u32, data: Vec<u8> },
}

// ProVerif constructor
#[hax_lib::pv_constructor]
fn create_message() -> Messages { /* ... */ }

// Process specifications
#[hax_lib::process_init]
fn initialize() {/* ... */}

#[hax_lib::process_read]
fn read_message() {/* ... */}

#[hax_lib::process_write]
fn write_message() {/* ... */}
```

## Internal Markers (Reference)
These are `hax-lib` functions that users should never call directly. They are the expansion targets for the procedural macros, used by the hax-engine to find specifications in the AST.

| **Macro** | **Expands to (contains)** |
| --- | --- |
| `#[requires(...)]` | `::hax_lib::_internal_precondition(...)` |
| `#[ensures(...)]` | `::hax_lib::_internal_postcondition(...)` |
| `#[loop_invariant(...)]` | `::hax_lib::_internal_loop_invariant(...)` |
| `#[loop_decreases(...)]` | `::hax_lib::_internal_loop_decreases(...)` |

## Complete Example
For educational purposes, an example to solidify the reader's understanding of the `hax-lib` API would be useful: 
```rust
use hax_lib::*;

/// A verified integer division function
#[requires(dividend < 1000 && divisor > 0)]
#[ensures(|result| result == dividend / divisor)]
#[hax_lib::fstar::options("--z3rlimit 100")]

pub fn safe_divide(dividend: u32, divisor: u32) -> u32 {
   // Assumption for specification
   assume!(divisor != 0);

   // Convert to mathematical integers
   let math_dividend: Int = dividend.lift();
   let math_divisor: Int = divisor.lift();
   
   // Logical assertion
   assert_prop!(math_divisor > Int::from(0));

   // Perform division
   let result = dividend / divisor;
   
   // Verify property 
   assert!(result <= dividend); 

   // Inline backend code
   hax_lib::fstar!("assert (v $result <= v $dividend)");
   result

}


/// A function with a verified loop

pub fn sum_array(arr: &[u32]) -> u32 {
   let mut sum = 0u32; 

   #[loop_invariant(sum <= i * u32::MAX && i <= arr.len())]
   #[loop_decreases(arr.len() - i)]

   for i in 0..arr.len() {
       let old_sum = sum;
       sum = sum.saturating_add(arr[i]);
       assert!(sum >= old_sum);
   }
   sum
} 
```