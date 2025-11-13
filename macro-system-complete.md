# Hax Macro System - Complete Exhaustive Documentation

## Table of Contents

1. [Macro System Overview](#macro-system-overview)
2. [Assertion and Assumption Macros](#assertion-and-assumption-macros)
3. [Contract Specification Macros](#contract-specification-macros)
4. [Loop Specification Macros](#loop-specification-macros)
5. [Refinement Type Macros](#refinement-type-macros)
6. [Visibility and Extraction Control](#visibility-and-extraction-control)
7. [Backend-Specific Macros](#backend-specific-macros)
8. [Protocol Verification Macros](#protocol-verification-macros)
9. [Mathematical and Logical Macros](#mathematical-and-logical-macros)
10. [Internal Implementation](#internal-implementation)
11. [Macro Expansion Process](#macro-expansion-process)
12. [Complete Syntax Reference](#complete-syntax-reference)

---

## Macro System Overview

The `hax` macro system is the primary user-facing API for formal verification. It uses a combination of Rust's built-in macro systems to inject specifications, guide the verification engine, and control code extraction. This allows developers to write code that is simultaneously verifiable by hax and compilable as standard, idiomatic Rust.

The system consists of two primary components:
- **Declarative Macros** (`macro_rules!`) - Used for lightweight, syntax-based transformations, such as `assert!` and `assume!`. These are fast, simple, and integrate well with the Rust compiler.
- **Procedural Macros** (`proc_macro`) - Used for complex Abstract Syntax Tree (AST) transformations. These are required for attribute macros like `#[requires]` and `#[loop_invariant]`, which need to parse and rewrite entire function or loop bodies.

### Architecture
The macro expansion process is the first step in the `hax` pipeline. When Rust code is compiled (either by `cargo build` or by `hax`), these macros rewrite the code to insert the necessary hooks for verification or runtime checks.

```
┌─────────────────────────────────────────────────────────┐
│                    User Code                            │
│  #[requires(x > 0)]                                     │
│  fn foo(x: u32) { assert!(x > 0); }                     │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│              Macro Expansion Layer                       │
├──────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌──────────────────────┐           │
│  │ Procedural      │  │ Declarative Macros   │           │
│  │ Macros          │  │ (macro_rules!)       │           │
│  │ (syn + quote)   │  │                      │           │
│  └─────────────────┘  └──────────────────────┘           │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Transformed Code                           │
│  fn foo(x: u32) {                                       │
│    #[cfg(hax)] hax_lib::_internal_precondition(x > 0);  │
│    #[cfg(hax)] hax_lib::assert(x > 0);                  │
│    #[cfg(not(hax))] assert!(x > 0);                     │
│  }                                                      │
└─────────────────────────────────────────────────────────┘
```

### Macro Categories

| Category | Purpose | Implementation Type |
|----------|---------|-------------------|
| Assertions | Runtime & static checks | Declarative |
| Contracts | Function specifications | Procedural |
| Loops | Loop invariants & termination | Procedural |
| Refinements | Type refinements | Procedural |
| Visibility | Extraction control | Procedural |
| Backend | Backend-specific code | Both |
| Protocol | Protocol specifications | Procedural |
| Mathematical | Math specifications | Declarative |

---

## Assertion and Assumption Macros
These macros are used to state facts about the program's state. `assert!` macros must be proven by the verifier, while `assume!` macros are given to the verifier as facts.

### `assert!` Macro

**Purpose**: Generate a proof obligation in formal verification while maintaining a standard runtime check in Rust. This is the most common macro used in verified code.

#### Syntax
```rust
assert!(condition)
assert!(condition, "format string", args...)
```

#### Implementation
The `assert!` macro is a declarative `macro_rules!` that cleverly delegates to either the `hax` verification-only `assert` function (when `cfg(hax)` is set) or the standard `::core::assert!` macro (when `cfg(not(hax))` is set).

```rust
#[macro_export]
macro_rules! assert {
    ($cond:expr $(,)?) => {
        // `proxy_macro_if_not_hax!` expands to `::core::assert!` when not in hax,
        // and to `$crate::assert` when in hax.
        $crate::proxy_macro_if_not_hax!(::core::assert, $crate::assert, $cond)
    };
    ($cond:expr, $($arg:tt)+) => {
        // The formatting arguments are passed to `::core::assert!`
        // but are discarded by `$crate::assert`.
        $crate::proxy_macro_if_not_hax!(::core::assert,    $crate::assert, $cond, $($arg)+)
    };
}

#[doc(hidden)]
#[cfg(hax)]
pub fn assert(_formula: bool) {
    // This is a marker function.
    // The hax engine intercepts calls to this and generates
    // a proof obligation (a Verification Condition).
    // It has no runtime effect in the extracted backend code.
}

#[doc(hidden)]
#[cfg(not(hax))]
macro_rules! proxy_macro_if_not_hax {
    ($macro:path, $f:expr, $($arg:tt)*) => {
        // When compiled normally, just expand to the standard `assert!`.
        $macro!($($arg)*)
    };
}

#[doc(hidden)]
#[cfg(hax)]
macro_rules! proxy_macro_if_not_hax {
    ($macro:path, $f:path, $cond:expr) => {
        // When compiled with hax, expand to the marker function call.
        $f($cond)
    };
    ($macro:path, $f:path, $cond:expr, $($arg:tt)+) => {
        // Discard formatting arguments.
            $f($cond)
    };
}

```

#### Examples
```rust
// Simple assertion
// Generates a proof obligation that `x > 0`.
assert!(x > 0);

// With message
// Generates a proof obligation that `x < 100`.
// The message is used by `cargo test` but ignored by `hax`.
assert!(x < 100, "x must be less than 100, got {}", x);

// In verification context
fn verified_function(x: u32) {
    #[requires(x < u32::MAX - 1)]
    let y = x + 1;
    assert!(y > x); // This proof obligation is satisfied by the laws of arithmetic.
}
```

#### Extraction Behavior
- In `hax`: Becomes a proof obligation. The verifier must prove condition is always true.
- In Rust (`cargo build`): Becomes `::core::assert!`. It is a runtime check in debug builds and is removed in release builds.
- In Rust (`cargo test`): A runtime check that will cause a panic on failure.

### `assume!` Macro

**Purpose**: Introduce a logical assumption without generating a proof obligation. This is a "_free pass_" for the verifier, but it is **unsafe** and can lead to unsound proofs if the assumption is false. It is primarily used for trusting unverified code or simplifying complex proofs.

#### Syntax
```rust
assume!(proposition)
```

#### Implementation
This macro expands to a `hax` marker function when verifying, and expands to nothing (`()`) in a standard Rust build, as it has no runtime meaning.

```rust
#[cfg(hax)]
#[macro_export]
macro_rules! assume {
    ($formula:expr) => {
        // `::hax_lib::Prop::from` converts the boolean expression
        // into a logical proposition for the backend.
        $crate::assume(::hax_lib::Prop::from($formula))
    };
}

#[cfg(not(hax))]
#[macro_export]
macro_rules! assume {
    ($formula:expr) => {
        () // No-op in regular Rust, it has no runtime effect.
    };
}

#[doc(hidden)]
#[cfg(hax)]
pub fn assume(_formula: Prop) {
    // Marker function intercepted by the hax engine.
    // Tells the verifier to add this proposition to its
    // context of "known facts" from this point forward.
}
```

#### Examples
```rust
// Basic assumption
// Used to prevent a panic in code that is not yet verified.
assume!(y != 0);
let result = x / y; // Verifier assumes `y != 0` and proves this is safe.

// Complex assumption
// Trusting the result of a complex, unverified library.
let product = complex_math_library::multiply(x, y);
assume!(product == x * y);

// Assuming a property about a loop
// This is UNSAFE, but can be a placeholder for a real loop invariant.
assume!(forall(|i: usize| i < arr.len() ==> arr[i] >= 0));
```

### `assert_prop!` Macro

**Purpose**: Assert a pure logical proposition that may involve quantifiers (`forall`, `exists`) or other constructs that are not computable at runtime. This macro _only_ exists for the verifier and compiles to nothing in standard Rust.

#### Syntax
```rust
assert_prop!(logical_proposition)
```

#### Implementation
This macro is similar to `assume!`, but it calls `assert_prop` which generates a proof obligation. It always expands to nothing in `cfg(not(hax))`.

```rust
#[macro_export]
macro_rules! assert_prop {
    ($($arg:tt)*) => {
        {
            #[cfg(hax)]
            {
                $crate::assert_prop(::hax_lib::Prop::from($($arg)*));
            }
            #[cfg(not(hax))]
            {
                // No runtime equivalent - compile to nothing
            }
        }
    };
}

#[doc(hidden)]
#[cfg(hax)]
pub fn assert_prop(_formula: Prop) {
    // Marker function intercepted by hax.
    // Generates a proof obligation for a pure logical proposition.
}
```

#### Examples
```rust
// Quantified assertions
assert_prop!(forall(|x: u32| x >= 0));
assert_prop!(exists(|x: i32| x < 0));

// Implications
assert_prop!(implies(x > 0, x >= 1));

// Proving a logical relationship between two functions
let a = foo(x);
let b = bar(x);
assert_prop!(implies(a > 5, b < 10));

// Proving properties of a data structure
assert_prop!(
    forall(|i: usize|
        implies(i < arr.len(),
            // Asserts that for every element, there's a larger or equal element
            exists(|j: usize| j < arr.len() && arr[j] >= arr[i])
        )
    )
);
```

### `debug_assert!` Macro

**Purpose:** A standard Rust assertion that exists only in debug builds (`cargo build`) and is completely ignored by `hax`. It generates no proof obligations and is not extracted.

#### Syntax
```rust
debug_assert!(condition)
debug_assert!(condition, "message", args...)
```

#### Implementation
This macro is a thin wrapper around `::core::debug_assert!` that is proxied to `()` (nothing) when `cfg(hax)` is enabled, effectively making it invisible to the verifier.

```rust
#[macro_export]
macro_rules! debug_assert {
    ($($arg:tt)*) => {
        // `no` is a marker for `proxy_macro_if_not_hax`
        $crate::proxy_macro_if_not_hax!(::core::debug_assert, no, $($arg)*)
    };
}

// `proxy_macro_if_not_hax` implementation for `cfg(not(hax))`
#[doc(hidden)]
#[cfg(not(hax))]
macro_rules! proxy_macro_if_not_hax {
    ($macro:path, $f:expr, $($arg:tt)*) => {
        $macro!($($arg)*)
    };
    // The `no` variant
    ($macro:path, no, $($arg:tt)*) => {
        $macro!($($arg)*)
    };
}

// `proxy_macro_if_not_hax` implementation for `cfg(hax)`
#[doc(hidden)]
#[cfg(hax)]
macro_rules! proxy_macro_if_not_hax {
    // The `no` variant for `hax`
    ($macro:path, no, $($arg:tt)*) => {
        () // Completely removed in hax
    };
    // ... other variants for `assert!` ...
}
```

#### Examples
```rust
// Use this for expensive runtime checks that you
// do not want to prove or include in verification.
fn expensive_check(vec: &Vec<u32>) {
    // This check is O(n^2) and would be difficult to prove
    // or slow down verification. `debug_assert!` is perfect.
    debug_assert!(
        vec.iter().all(|x| vec.iter().all(|y| (x + y) % 2 == 0)),
        "All numbers must have the same parity"
    );
}
```

---

## Contract Specification Macros
These are procedural attribute macros that attach formal contracts (preconditions, postconditions, and lemmas) to Rust functions.

### `#[requires(...)]` Attribute Macro

**Purpose:** Specify one or more _preconditions_ for a function. The caller of the function must prove that these conditions hold. The function body can then _assume_ they hold.

#### Syntax
```rust
#[requires(condition)]
#[requires(condition1)]
#[requires(condition2)] // Multiple requires are conjunctive (ANDed together)
fn my_func(arg: Type) { ... }
```

#### Implementation
This is a procedural macro that parses the function it's attached to. It injects a special `_internal_precondition` call at the top of the function body (for `hax` to see) and also adds a `#[cfg_attr]` attribute for the `hax` extraction engine to read directly.

```rust
// hax-lib-macros/src/lib.rs
#[proc_macro_attribute]
pub fn requires(attr: TokenStream, item: TokenStream) -> TokenStream {
    let predicate = parse_macro_input!(attr as Expr);
    let mut function = parse_macro_input!(item as ItemFn);
    
    // 1. Insert precondition check at function start for hax engine
    let precondition_check = quote! {
        #[cfg(hax)]
        ::hax_lib::_internal_precondition(#predicate);
    };
    
    function.block.stmts.insert(0, parse_quote!(#precondition_check));
    
    // 2. Add an attribute for the hax extraction process
    function.attrs.push(parse_quote!(
        #[cfg_attr(hax, ::hax_lib::contract::requires(#predicate))]
    ));
    
    TokenStream::from(quote!(#function))
}
```

#### Examples
```rust
// Single precondition
#[requires(x > 0.0)]
fn safe_sqrt(x: f64) -> f64 {
    x.sqrt() // Verifier can assume x > 0.0
}

// Multiple preconditions
#[requires(x < 100)]
#[requires(y < 100)]
fn bounded_add(x: u32, y: u32) -> u32 {
    // Verifier can assume x < 100 AND y < 100
    let sum = x + y;
    assert!(sum < 200); // This is provable
    sum
}

// With complex expressions
#[requires(arr.len() > 0)]
#[requires(index < arr.len())]
fn safe_access(arr: &[u32], index: usize) -> u32 {
    // Verifier can assume the array is not empty AND the index is in bounds.
    arr[index] // This index is proven to be safe.
}

// Using mathematical integers (`Int`) for overflow safety
#[requires(Int::from(x) + Int::from(y) <= Int::from(u32::MAX))]
fn no_overflow_add(x: u32, y: u32) -> u32 {
    x + y // This addition is proven not to overflow.
}
```

### `#[ensures(|result| ...)]` Attribute Macro

**Purpose:** Specify one or more _postconditions_ for a function. The function body must prove that these conditions hold upon returning. The caller can then _assume_ they hold.

#### Syntax
```rust
#[ensures(|result| condition_on_result)]
#[ensures(|res| condition_on_res && condition_on_args)] // Custom name for result
fn my_func(arg: Type) -> RetType { ... } 
```
The closure argument (`result`, `res`) is the name given to the return value of the function.

#### Implementation
This macro is more complex. It wraps the entire function body in a new block, captures the return value, and inserts an `_internal_postcondition` check after the original body has executed but _before_ the value is returned.

```rust
// In hax-lib-macros/src/lib.rs (simplified)
use syn::ExprClosure;

#[proc_macro_attribute]
pub fn ensures(attr: TokenStream, item: TokenStream) -> TokenStream {
    let postcondition = parse_macro_input!(attr as ExprClosure);
    let mut function = parse_macro_input!(item as ItemFn);

    // 1. Extract the name for the result (e.g., `result` from `|result| ...`)
    let result_param = &postcondition.inputs[0];
    let postcondition_body = &postcondition.body;

    // 2. Wrap the original function body
    let original_body = &function.block;
    let new_body = quote! {
        {
            // Execute the original body and capture its return value
            let #result_param = #original_body;

            #[cfg(hax)]
            {
                // The `let #result_param = #result_param;` is a trick
                // to make the variable available inside the prop.
                let #result_param = #result_param;
                ::hax_lib::_internal_postcondition(
                    ::hax_lib::Prop::from(#postcondition_body)
                );
            }
            // Return the captured result
            #result_param
        }
    };
    // `parse_quote!` converts the `quote!` tokens back into a `Block`
    function.block = parse_quote!(#new_body);

    // 3. Add attribute for extraction
    function.attrs.push(parse_quote!(
        #[cfg_attr(hax, ::hax_lib::contract::ensures(#postcondition))]
    ));
    
    TokenStream::from(quote!(#function))
}
```

#### Examples
```rust
// Basic postcondition
#[ensures(|result| result > 0)]
fn positive_result() -> i32 {
    42 // Proof obligation: `assert!(42 > 0)`, which is true.
}

// Relating result to input
#[requires(x < u32::MAX)]
#[ensures(|result| result > x)]
fn increment(x: u32) -> u32 {
    x + 1 // Proof obligation: `assert!(x + 1 > x)`, which is true.
}

// Multiple postconditions
#[ensures(|result| result.len() == arr.len())]
#[ensures(|result| forall(|i: usize| i < result.len() ==> result[i] == arr[i] * 2))]
fn double_array(arr: &[u32]) -> Vec<u32> {
    arr.iter().map(|x| x * 2).collect() // Proof obligation: prove both postconditions
}

// With Option type
#[ensures(|result| result.is_some() <==> y != 0)]
#[ensures(|result| implies(result.is_some(), result.unwrap() == x / y))]
fn safe_divide(x: u32, y: u32) -> Option<u32> {
    if y == 0 {
        None // Case 1: y == 0. `result.is_some()` is false. Both `ensures` hold.
    } else {
        Some(x / y) // Case 2: y != 0. `result.is_some()` is true. Both `ensures` hold.
    }
}
```

### `#[lemma]` Attribute Macro

**Purpose:** Define a "ghost function" or "proof-only function." A lemma has no runtime behavior and is _erased_ during standard Rust compilation. In `hax`, it is treated as a theorem to be proven. Its `#[ensures(...)]` clause is a property that, once proven, can be _assumed_ by any function that calls the lemma.

#### Implementation
This macro is a procedural attribute that rewrites the function to be `#[inline(always)] fn ... {}` (an empty body) for `cfg(not(hax))`, and adds the necessary `hax` attributes for extraction.

```rust
// In hax-lib-macros/src/lib.rs
#[proc_macro_attribute]
pub fn lemma(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let function = parse_macro_input!(item as ItemFn);
    let sig = &function.sig;

    quote! {
        // This is the "real" function that hax will extract and verify.
        // It includes the body, which contains the proof steps (asserts).
        #[cfg(hax)]
        #[cfg_attr(hax, ::hax_lib::contract::lemma)]
        #function

        // This is the "fake" function for standard Rust builds.
        // It has the same signature but an empty body, so the
        // compiler optimizes away any calls to it.
        #[cfg(not(hax))]
        #[inline(always)]
        #sig {
            // No runtime code
        }
    }
}
```

#### Examples
```rust
// Lemma proving a simple arithmetic property
#[lemma]
#[ensures(x + y == y + x)]
fn add_is_commutative(x: u32, y: u32) {
    // The proof is trivial and can be an empty body
    // or contain `assert_prop!` for the solver.
    assert_prop!(x + y == y + x);
}

// A function that *uses* the lemma
fn complex_calc(a: u32, b: u32) {
    let val1 = a + b;
    
    // Call the lemma. This has no runtime cost, but
    // in the verifier, it adds the fact `a + b == b + a`
    // to the context.
    add_is_commutative(a, b);

    let val2 = b + a;
    assert!(val1 == val2); // This is now provable thanks to the lemma
}

// A lemma with a proof body
#[lemma]
#[requires(x < u32::MAX / 2 && y < u32::MAX / 2)]
#[ensures(x + y < u32::MAX)]
fn sum_is_safe(x: u32, y: u32) {
    // This assertion *is* the proof
    assert!(x + y < u32::MAX / 2 + u32::MAX / 2);
    assert!(x + y < u32::MAX);
}
```

---

## Loop Specification Macros
Loops are a primary source of complexity in verification. These macros allow you to specify loop invariants (properties that are true on every iteration) and termination measures (proofs that the loop will eventually end).

### `#[loop_invariant(...)]` Attribute Macro

**Purpose:** Specify one or more invariants for a `for`, `while`, or `loop` expression. The verifier must prove that the invariant holds before the loop begins, and that _every_ iteration of the loop preserves the invariant.

#### Syntax
```rust
#[loop_invariant(condition)]
for var in iterator { ... }

#[loop_invariant(condition1)]
#[loop_invariant(condition2)]
while condition { ... }

#[loop_invariant(condition)]
loop { ... }
```

#### Implementation
This procedural macro is complex because it must parse different loop types (`for`, `while`, `loop`). It injects internal `_loop_invariant` check functions inside the loop body for the `hax` engine to detect.

```rust
// In hax-lib-macros/src/lib.rs (simplified)
use syn::{Expr, ExprForLoop, ExprWhile};

#[proc_macro_attribute]
pub fn loop_invariant(attr: TokenStream, item: TokenStream) -> TokenStream {
    let invariant = parse_macro_input!(attr as Expr);
    let loop_expr_input = parse_macro_input!(item as Expr);
    
    let check = quote! {
        #[cfg(hax)]
        ::hax_lib::_internal_loop_invariant(::hax_lib::Prop::from(#invariant));
    };
    
    let loop_with_invariant = match loop_expr_input {
        Expr::ForLoop(mut for_loop) => {
            // Insert invariant check at loop *start*
            for_loop.body.stmts.insert(0, parse_quote!(#check));
            
            // Add attribute for extraction
            for_loop.attrs.push(parse_quote!(
                #[cfg_attr(hax, ::hax_lib::loop_spec::invariant(#invariant))]
            ));
            
            quote!(#for_loop)
        }
        Expr::While(mut while_loop) => {
            // Insert invariant check at loop *start*
            while_loop.body.stmts.insert(0, parse_quote!(#check));
            
            // Add attribute for extraction
            while_loop.attrs.push(parse_quote!(
                #[cfg_attr(hax, ::hax_lib::loop_spec::invariant(#invariant))]
            ));

            quote!(#while_loop)
        }
        Expr::Loop(mut loop_expr) => {
            // Insert invariant check at loop *start*
            loop_expr.body.stmts.insert(0, parse_quote!(#check));
            
            // Add attribute for extraction
            loop_expr.attrs.push(parse_quote!(
                #[cfg_attr(hax, ::hax_lib::loop_spec::invariant(#invariant))]
            ));

            quote!(#loop_expr)
        }

            quote!(#loop_expr)
        }
        => panic!("loop_invariant can only be applied to for, while, or loop expressions"),
    };

    TokenStream::from(loop_with_invariant)
}

```

#### Examples
```rust
// For loop with invariant
let mut sum = 0;
let mut i = 0;
#[loop_invariant(sum == old(sum) + arr[old(i)])] // This is hard to state
#[loop_invariant(sum == arr[..i].iter().sum())] // This is the better invariant
#[loop_invariant(i <= arr.len())]
while i < arr.len() {
    sum += arr[i];
    i += 1;
}
assert!(sum == arr.iter().sum()); // Provable from invariant

// While loop with invariant
let mut x = n;
let mut result = 1;
#[loop_invariant(result * factorial(x) == factorial(n))]
while x > 0 {
    result *= x;
    x -= 1;
}
assert!(result == factorial(n)); // Provable

// Binary search invariants
let mut left = 0;
let mut right = arr.len();
#[loop_invariant(left <= right && right <= arr.len())]
#[loop_invariant(
    forall(|j: usize| 
        implies(j < left, arr[j] < target) &&
        implies(j >= right && j < arr.len(), arr[j] >= target)
    )
)]
while left < right {
    let mid = left + (right - left) / 2;
    if arr[mid] < target {
        left = mid + 1;
    } else {
        right = mid; // Not mid - 1, to preserve the invariant
    }
}
assert!(left == right);
// From invariant, we know `left` is the insertion point
```

### `#[decreases(...)]` and `#[loop_decreases(...)]` Macros

**Purpose:** Specify a "_termination measure_" for a recursive function (`#[decreases]`) or a loop (`#[loop_decreases]`). This is a value that is guaranteed to get smaller with each recursive call or loop iteration, and which cannot decrease forever (e.g., a natural number `u32` decreasing to `0`). This proves that the function or loop will not run forever.

#### Syntax
```rust
#[decreases(expression)]
fn recursive_function(...) { ... }

#[loop_decreases(expression)]
while condition { ... }
```

#### Implementation
These macros are attributes that simply add a `cfg_attr` for the `hax` extractor to read. The logic of checking the decrease is handled entirely by the verification backend (e.g., `F*` or `Lean4`).

```rust
// In hax-lib-macros/src/lib.rs (simplified)

#[proc_macro_attribute]
pub fn decreases(attr: TokenStream, item: TokenStream) -> TokenStream {
    let measure = parse_macro_input!(attr as Expr);
    let mut function = parse_macro_input!(item as ItemFn);
    
    // Add decreases check for recursive calls
    let fn_name = &function.sig.ident;
    let check = quote! {
        #[cfg(hax)]
        {
            static DECREASES_STACK: ::std::cell::RefCell<Vec<_>> = 
                ::std::cell::RefCell::new(Vec::new());
            
            let current_measure = #measure;
            
            if let Some(prev_measure) = DECREASES_STACK.borrow().last() {
                ::hax_lib::_internal_check_decreases(prev_measure, &current_measure);
            }
            
            DECREASES_STACK.borrow_mut().push(current_measure);
            
            // Original function body here
            
            DECREASES_STACK.borrow_mut().pop();
        }
    };
    
    // Add attribute for extraction
    function.attrs.push(parse_quote!(
        #[cfg_attr(hax, ::hax_lib::termination::decreases(#measure))]
    ));
    
    TokenStream::from(quote!(#function))
}

#[proc_macro_attribute]
pub fn loop_decreases(attr: TokenStream, item: TokenStream) -> TokenStream {
    let measure = parse_macro_input!(attr as Expr);
    let loop_expr = parse_macro_input!(item as Expr);
    
    match loop_expr {
        Expr::While(mut while_loop) => {
            // Add decreases check in loop
            let check = quote! {
                #[cfg(hax)]
                {
                    let loop_measure = #measure;
                    ::hax_lib::_internal_loop_decreases(
                        ::hax_lib::Int::from(loop_measure)
                    );
                }
            };
            
            while_loop.body.stmts.insert(0, parse_quote!(#check));
            
            TokenStream::from(quote!(#while_loop))
        }
        _ => panic!("loop_decreases can only be applied to while loops"),
    }
}
```

#### Examples
```rust
// Recursive function with decreases
#[decreases(n)]
fn factorial(n: u32) -> u64 {
    if n == 0 {
        1
    } else {
        // `hax` proves that `n - 1` is strictly less than `n`.
        (n as u64) * factorial(n - 1)
    }
}

// Mutual recursion with lexicographic ordering
#[decreases((n, 0))] // `(n, 0)` is the measure
fn even(n: u32) -> bool {
    if n == 0 {
        true
    } else {
        // `hax` proves `(n - 1, 1)` < `(n, 0)`
        odd(n - 1)
    }
}

#[decreases((n, 1))] // `(n, 1)` is the measure
fn odd(n: u32) -> bool {
    if n == 0 {
        false
    } else {
        // `hax` proves `(n - 1, 0)` < `(n, 1)`
        even(n - 1)
    }
}

// Loop with decreases
let mut i = n;
#[loop_decreases(i)]
while i > 0 {
    process(i);
    i -= 1; // `hax` proves that `i` decreases in each iteration.
}

// For loops (termination is usually proven automatically)
// but can it be made explicit, if desired. 
#[loop_decreases(arr.len() - i)]
for i in 0..arr.len() {
    // ...
}

// Complex termination measure
let mut x = 100;
let mut y = 50;
#[loop_decreases(x + y * 2)]
while x > 0 || y > 0 {
    if y > 0 {
        y -= 1;
    } else {
        x -= 1;
    }
}
```

---

## Refinement Type Macros
This system allows you to create new Rust types that "refine" a base type with a logical invariant (a predicate). This makes the invariant part of the type itself, enabling stronger type-checking at the verification level.

### `#[refinement_type]` Attribute Macro

**Purpose**: Define a struct that acts as a refinement type. It automatically generates the boilerplate Refinement trait implementation.

#### Syntax
```rust
#[refinement_type]
struct TypeName {
    #[refinement]
    field: BaseType,
    // ... other (ghost) fields
}
// Must be paired with a manual `impl Refinement for TypeName`
impl Refinement for TypeName {
    fn invariant(value: BaseType) -> Prop {
        // The predicate that must hold
        Prop::from(condition)
    }
}
}
```

#### Implementation
This macro is a stub that generates the trait boilerplate. It requires the user to manually impl Refinement to provide the actual invariant.

```rust
// In hax-lib-macros/src/lib.rs (simplified)
use syn::{ItemStruct, Fields};

#[proc_macro_attribute]
pub fn refinement_type(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let input = parse_macro_input!(item as ItemStruct);
    let name = &input.ident;
    
    // Find the refinement field
    let refinement_field = input.fields.iter()
        .find(|f| f.attrs.iter().any(|a| a.path.is_ident("refinement")))
        .expect("refinement_type requires one field marked with #[refinement]");

    let field_name = &refinement_field.ident;
    let field_type = &refinement_field.ty;
    
    // Generate the Refinement trait implementation boilerplate
    let gen = quote! {
        #input // Re-emit the struct definition

        impl ::hax_lib::Refinement for #name {
            type InnerType = #fields;

            // `new` is the "unsafe" constructor.
            // It assumes the invariant holds.
            fn new(x: Self::InnerType) -> Self {
                // At runtime, we can add a debug check
                #[cfg(debug_assertions)]
                {
                    if !<Self as ::hax_lib::Refinement>::invariant(x.clone()).check() {
                        panic!("Refinement invariant violated for {}", stringify!(#name));
                    }
                }
                
                #name {
                    #field_name: x,
                    // ... other fields must be ghost or `Default`
                }
            }
            
            fn get(self) -> Self::InnerType {
                self.#field_name
            }
            
            fn get_ref(&self) -> &Self::InnerType {
                &self.#field_name
            }
            
            // The user MUST override this function in a manual impl block
            fn invariant(value: Self::InnerType) -> ::hax_lib::Prop {
                ::hax_lib::Prop::from(true) // Default is `true`
            }
        }
        

        // Auto-implement `Deref` for convenience
        impl ::core::ops::Deref for #name {
            type Target = #field_type;
            fn deref(&self) -> &Self::Target {
                &self.#field_name
            }
        }
    };

    TokenStream::from(gen)
}
```

#### Examples
```rust
// Define a non-zero integer type
#[refinement_type]
pub struct NonZeroU32 {
    #[refinement]
    value: u32,
}

// Manually provide the invariant
impl ::hax_lib::Refinement for NonZeroU32 {
    fn invariant(value: u32) -> ::hax_lib::Prop {
        ::hax_lib::Prop::from(value != 0)
    }
}

// We can now write a "smart constructor" that proves the invariant
impl NonZeroU32 {
    #[ensures(|result| result.is_some() <==> value != 0)]
    #[ensures(|result| implies(result.is_some(), result.unwrap().value == value))]
    pub fn new_checked(value: u32) -> Option<Self> {
        if value != 0 {
            // `new` is the "unsafe" constructor from the trait
            Some(Self::new(value))
        } else {
            None
        }
    }
}

// This function can now safely assume its input is not zero
#[requires(y.value != 0)] // This is redundant due to the type
fn verified_divide(x: u32, y: NonZeroU32) -> u32 {
    // We can access the inner value via `deref`
    assert!(*y != 0);
    x / *y // This is proven safe
}
```

```rust
// Define a type for even numbers
#[refinement_type]
pub struct Even {
    #[refinement]
    value: u32,
}

impl ::hax_lib::Refinement for Even {
    fn invariant(value: u32) -> ::hax_lib::Prop {
        ::hax_lib::Prop::from(value % 2 == 0)
    }
}

// This function's signature *proves* it returns an even number
#[ensures(|result| result.value == x * 2)]
fn double_it(x: u32) -> Even {
    let val = x * 2;
    assert!(val % 2 == 0); // Prove the invariant
    Even::new(val) // Construct the refined type
}
```

---

## Visibility and Extraction Control
These macros control how the `hax` engine "sees" your Rust code. They allow you to hide complex implementations, exclude irrelevant code, or force the inclusion of helper items.

### `#[include]` Attribute Macro

**Purpose**: Force inclusion of an item (function, struct, etc.) in the `hax` extraction, even if it's not public or directly called by another verified item. This is useful for including `#[lemma]` functions or helper types.

#### Implementation
This is a simple attribute macro that adds a `cfg_attr` marker, which the `hax` extractor is configured to find.

```rust
// In hax-lib-macros/src/lib.rs (simplified)
use syn::Item;

#[proc_macro_attribute]
pub fn include(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let mut item = parse_macro_input!(item as Item);
    
    // Add marker attribute for extraction
    let attrs = match &mut item {
        Item::Fn(f) => &mut f.attrs,
        Item::Struct(s) => &mut s.attrs,
        Item::Enum(e) => &mut e.attrs,
        Item::Type(t) => &mut t.attrs,
        Item::Impl(i) => &mut i.attrs,
        Item::Trait(t) => &mut t.attrs,
        _ => panic!("#[include] can only be applied to items"),
    };
    
    attrs.push(parse_quote!(
        #[cfg_attr(hax, ::hax_lib::extraction::include)]
    ));
    
    TokenStream::from(quote!(#item))
}
```

#### Examples
```rust
#[include] // Ensure this lemma is extracted even if unused
#[lemma]
#[ensures(x + y == y + x)]
fn add_is_commutative(x: u32, y: u32) {
    assert_prop!(x + y == y + x);
}

#[include] // Ensure this helper type is available in the backend
struct HelperProofState {
    val: u32,
}
```

### `#[exclude]` Attribute Macro

**Purpose**: Explicitly _exclude_ an item from the hax extraction. This is useful for test harnesses, debugging functions, or platform-specific code that should not be part of the formal verification.

#### Implementation
This macro adds an `extraction::exclude` attribute for `hax` and `allow(dead_code)` for `not(hax)` (since the verified code might not call it, and we don't want cargo build to warn).

```rust
// In hax-lib-macros/src/lib.rs (simplified)
#[proc_macro_attribute]
pub fn exclude(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let item = parse_macro_input!(item as Item);

    quote! {
        // This attribute tells the hax extractor to ignore this item.
        #[cfg_attr(hax, ::hax_lib::extraction::exclude)]
        // This attribute suppresses warnings if the item isn't used
        // in the `not(hax)` build.
        #[cfg_attr(not(hax), allow(dead_code))]
        #item
    }
}
```

#### Examples
```rust
#[exclude]
fn debug_print_state(state: &MyState) {
    // This function is full of `println!` and other
    // non-verifiable code. We exclude it completely.
    println!("{:#?}", state);
}

#[exclude]
#[cfg(test)]
mod tests {
    // Exclude the entire test module from verification.
    #[test]
    fn my_test() { ... }
}
```

### `#[opaque]` and `#[opaque_type]` Macros

**Purpose**: Hide the _implementation_ (body) of a function or type from the verifier, exposing only its _signature_ and contracts. This is a crucial technique for abstraction and modularity. The verifier will _assume_ the function's `#[ensures]` contract holds, without checking its body.

#### Implementation
`#[opaque_type]` is a simple marker attribute. `#[opaque]` is more complex: it replaces the function body with a `hax`-specific marker `opaque_call` when verifying, but keeps the original implementation for standard Rust builds.

```rust
// In hax-lib-macros/src/lib.rs (simplified)
#[proc_macro_attribute]
pub fn opaque(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let mut function = parse_macro_input!(item as ItemFn);
    
    let original_body = &function.block;
    let sig_str = quote!(stringify!(#function.sig)).to_string();

    // Generate a new body that branches on `cfg(hax)`
    let new_body = quote! {
        {
            #[cfg(hax)]
            {
                // Hax sees only this: an abstract call.
                // It will assume the `#[ensures]` contract holds.
                ::hax_lib::opaque_call(#sig_str)
            }
            
            #[cfg(not(hax))]
            {
                // Standard Rust sees the original implementation.
                #original_body
            }
        }
    };
    
    function.block = parse_quote!(#new_body);
    
    TokenStream::from(quote!(
        // Add the marker attribute for the extractor
        #[cfg_attr(hax, ::hax_lib::extraction::opaque)]
        #function
    ))
}

#[proc_macro_attribute]
pub fn opaque_type(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let item = parse_macro_input!(item as Item);

    quote! {
        // Simple marker for type definitions
        #[cfg_attr(hax, ::hax_lib::extraction::opaque_type)]
        #item
    }
}
```

#### Examples
```rust
// Make a complex, optimized function opaque.
#[opaque]
#[requires(x < 1_000_000)]
#[ensures(|result| result == (x as u64 * x as u64) % 1337)]
fn complex_hash(x: u32) -> u64 {
    // This implementation is highly optimized, complex,
    // and irrelevant to the calling function's logic.
    // We verify it *once*, then make it `#[opaque]`.
    let mut val = (x as u64).wrapping_mul(37);
    val = (val >> 3) | (val << 61);
    val % 1337
}

// Another function can now call `complex_hash`
// and its proof will be *fast*.
fn use_hash(y: u32) {
    #[requires(y < 1_000_000)]
    let h = complex_hash(y);
    // The verifier *assumes* the contract of `complex_hash`
    // and knows that `h == (y * y) % 1337`.
    assert!(h < 1337); // This is provable.
}

// Make a type's internal representation abstract
#[opaque_type]
struct MyCryptoKey {
    // The verifier will not know that `MyCryptoKey`
    // is just a wrapper around `[u8; 32]`.
    // It will be treated as a new, abstract type.
    internal: [u8; 32],
}
```

---

## Backend-Specific Macros
These macros provide an "escape hatch" to interact directly with the verification backend (e.g., `F*`, `Lean4`, `Coq`). They are powerful but non-portable and should be used sparingly.

### F* Specific Macros
These macros are used when `hax` is targeting the `F*` verification system.

#### `fstar!` Macro

**Purpose**: Inject raw `F*` code directly into the extracted module. This is most often used to call `F*` lemmas from its standard library (`FStar.Math.Lemmas`, etc.) to help the `SMT` solver.

#### Syntax
```rust
hax_lib::fstar!("F* code here");
hax_lib::fstar!(r#"
    multi-line
    F* code
"#);
```

#### Implementation
This is a `macro_rules!` macro that expands to a `backend::fstar::inline` marker function, which the `hax` extractor intercepts. It compiles to nothing in standard Rust.

```rust
#[macro_export]
macro_rules! fstar {
    ($code:expr) => {
        #[cfg(hax)]
        {
            // This function is a special marker that the
            // hax extractor recognizes and replaces with
            // the raw F* code.
            ::hax_lib::backend::fstar::inline($code)
        }
        // Note: No `cfg(not(hax))` branch, so it
        // compiles to nothing in standard Rust.
    };
}

#[cfg(hax)]
pub mod backend {
    pub mod fstar {
        #[doc(hidden)]
        pub fn inline(_code: &str) {
            // Marker function for extraction
        }
    }
}
```

#### Examples
```rust
fn complex_proof(x: u32, y: u32) {
    #[requires(x > 0 && y > 0)]
    #[ensures(x * y / x == y)]

    // We need to help the SMT solver with non-linear arithmetic.
    // We can call an F* lemma to provide the proof.
    hax_lib::fstar!(r#"
        // Call a lemma from the F* standard library
        FStar.Math.Lemmas.lemma_div_mult $x $y $x;
    "#);
    
    let z = (x * y) / x;
    assert!(z == y); // This is now provable
}

fn use_fstar_lemma(x: u32, y: u32) {
    hax_lib::fstar!(r#"
        // Manually assert properties about bitwise operations
        FStar.UInt32.lemma_add_mod $x $y;
    "#);
}
```

### `#[fstar::options(...)]` Attribute

**Purpose**: Set `F*` command-line options for a specific item (e.g., function or module). This allows you to increase the `SMT` solver timeout (`z3rlimit`), fuel, or `ifuel` for a particularly difficult proof, or to `admit` a proof entirely.



#### Implementation
This is a procedural attribute macro that adds a `cfg_attr` marker for the `hax` extractor.

```rust
// In hax-lib-macros/src/lib.rs (simplified)
use syn::{LitStr, Item};

#[proc_macro_attribute]
pub fn fstar_options(attr: TokenStream, item: TokenStream) -> TokenStream {
    let options = parse_macro_input!(attr as LitStr);
    let item = parse_macro_input!(item as Item);
    
    quote! {
        #[cfg_attr(hax, ::hax_lib::backend::fstar::options(#options))]
        #item
    }
}
```

#### Examples
```rust
// Give the solver 20 seconds (20000 * 10ms) and more fuel
#[hax_lib::fstar::options("--z3rlimit 20000 --fuel 4 --ifuel 2")]
fn very_complex_function_proof() {
    // ...
}

// Admit (skip) the proof for this function
#[hax_lib::fstar::options("--admit_smt_queries true")]
fn assumed_correct() {
    // This function's contracts are assumed to be true.
}
```

---

## Protocol Verification Macros
This is an advanced feature set for verifying communication protocols, often modeled as state machines.

### `#[protocol_messages]` Attribute

**Purpose**: Define an `enum` as the set of all possible messages in a protocol. This macro can auto-generate serialization/deserialization logic (`send`/`receive`) and link the enum to the state machine verification logic.

#### Implementation
```rust
// In hax-lib-macros/src/lib.rs (simplified)
use syn::ItemEnum;

#[proc_macro_attribute]
pub fn protocol_messages(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let mut enum_def = parse_macro_input!(item as ItemEnum);

    // Add protocol-specific derives and attributes
    enum_def.attrs.push(parse_quote!(
        // Add derives for serialization, debugging, etc.
        #[derive(Clone, Debug, PartialEq, ::hax_lib::protocol::Serialize, ::hax_lib::protocol::Deserialize)]
    ));

    // Generate send/receive helper functions
    let enum_name = &enum_def.ident;
    let send_recv = quote! {
        impl #enum_name {
        
            // This function is verified to correctly serialize and send
            #[ensures(/* ... */)]
            pub fn send(&self, channel: &mut Channel) {
                ::hax_lib::protocol::send_message(channel, self)
            }

            // This function is verified to correctly receive and deserialize
            #[ensures(/* ... */)]
            pub fn receive(channel: &mut Channel) -> Option<Self> {
                ::hax_lib::protocol::receive_message(channel)
            }
        }
    };

    TokenStream::from(quote! {
        #enum_def
        #send_recv
    })
}

#### Examples
#[protocol_messages]
enum TlsMessage {
    ClientHello(ClientHelloPayload),
    ServerHello(ServerHelloPayload),
    AppData(Vec<u8>),
    Alert(AlertPayload),
}

// The macro generates:
// impl TlsMessage {
//   pub fn send(&self, ...) { ... }
//   pub fn receive(channel: &mut Channel) -> Option<Self> { ... }
// }
//
// And derives `Serialize` and `Deserialize`.
```

### `#[state_machine]` Attribute Macro

**Purpose:** Define and verify a state machine. This macro attaches to a struct representing the machine's state and an `impl` block containing the transition functions. `hax` will prove that all transitions are valid and that the machine's invariants hold.

#### Implementation

This is one of the most complex macros. It rewrites the `impl` block to thread a ghost state variable through all transitions, proving that each transition correctly follows the specified protocol logic.

```rust
// In hax-lib-macros/src/lib.rs (simplified)
#[proc_macro_attribute]
pub fn state_machine(attr: TokenStream, item: TokenStream) -> TokenStream {
    // 1. Parse the protocol definition (e.g., `#[state_machine(MyProtocol)]`)
    let protocol_name = parse_macro_input!(attr as Ident);
    
    // 2. Parse the `impl` block
    let mut impl_block = parse_macro_input!(item as ItemImpl);

    // 3. For each function in the `impl` block:
    for item in &mut impl_block.items {
        if let ImplItem::Fn(func) = item {
            // 4. Add `#[requires]` based on protocol:
            // e.g., "this transition is only valid in `State::Handshake`"
            func.attrs.push(parse_quote!(
                #[requires(self.state.can_transition_to( ... ))]
            ));

            // 5. Add `#[ensures]` based on protocol:
            // e.g., "after this, the state is `State::AppData`"
            func.attrs.push(parse_quote!(
                #[ensures(|_| self.state == ...)]
            ));
            
            // 6. Inject ghost code to update the abstract protocol state
        }
    }
    
    TokenStream::from(quote!(#impl_block))
}
```

#### Examples
```rust
// 1. Define the abstract protocol states (ghost code)
#[ghost]
enum TlsProtocolState {
    Start,
    ClientHelloSent,
    ServerHelloReceived,
    AppData,
    Closed,
}

// 2. Define the concrete state struct
struct TlsConnection {
    state: TlsProtocolState,
    session_key: Option<Key>,
    // ... other fields
}

// 3. Apply the macro to the `impl` block
#[state_machine(TlsProtocolState)]
impl TlsConnection {
    // The macro verifies this transition
    #[transition(TlsProtocolState::Start => TlsProtocolState::ClientHelloSent)]
    pub fn send_client_hello(&mut self) {
        // ... send the message
        // The macro implicitly asserts the state is now `ClientHelloSent`
    }

    // The macro verifies this transition
    #[transition(TlsProtocolState::ClientHelloSent => TlsProtocolState::ServerHelloReceived)]
    pub fn receive_server_hello(&mut self, msg: ServerHelloPayload) {
        // ... process message, derive key
        self.session_key = Some(derive_key(msg));
    }
}
```

## Mathematical and Logical Macros

These are `macro_rules!` macros that provide a Rust-like syntax for pure logical propositions used in contracts.

### `forall!` and `exists!` Macros

**Purpose:** Express universal (`∀`) and existential (`∃`) quantification.

```rust
// `forall!(|x: T, y: T| ...)`
#[macro_export]
macro_rules! forall { ... }

// `exists!(|x: T, y: T| ...)`
#[macro_export]
macro_rules! exists { ... }

// Example
assert_prop!(
    forall(|x: u32| implies(x > 0, exists(|y: u32| y < x)))
);
```

### `implies!` Macro
**Purpose:** Express logical implication (A ==> B).

```rust
#[macro_export]
macro_rules! implies {
    ($a:expr, $b:expr) => {
        ::hax_lib::Prop::Implies(Box::new($a.into()), Box::new($b.into()))
    };
}

// Example
#[requires(implies(user.is_admin, user.has_key))]
fn admin_action(user: &User) { ... }
```

### `Int` and `Real` Types

**Purpose:** Provide access to mathematical (unbounded) integers (`Int`) and reals (`Real`) for specifications, distinguishing them from machine integers (`u32`, `i64`).

```rust
// Example: Proving overflow safety
#[requires(Int::from(x) + Int::from(y) <= Int::from(u32::MAX))]
fn safe_add(x: u32, y: u32) -> u32 {
    x + y
}

// Example: Proving float properties
#[requires(Real::from(x) > 0.0)]
fn safe_log(x: f64) -> f64 {
    x.log2()
}
```

## Internal Implementation
This section describes the internal "marker" functions and traits that the macros expand into. **Users should never call these directly**.

- `::hax_lib::_internal_precondition(Prop)`: Marker function inserted by `#[requires]`.

- `::hax_lib::_internal_postcondition(Prop)`: Marker function inserted by `#[ensures]`.

- `::hax_lib::_internal_loop_invariant(Prop)`: Marker function inserted by `#[loop_invariant]`.

- `::hax_lib::contract::requires`, `::hax_lib::contract::ensures`, ...: Trait-based attributes read by the `hax` extractor.

- `::hax_lib::Prop`: The internal struct used to build logical propositions (e.g., `Prop::Implies(...)`, `Prop::Forall(...)`). The `From<bool>` trait is implemented for `Prop` to convert simple boolean expressions.

- `::hax_lib::Refinement`: The trait automatically implemented by `#[refinement_type]`.

## Macro Expansion Process

1. User writes code:
```rust
#[requires(x > 0)]
fn my_func(x: u32) { assert!(x != 0); }
```

2. `cargo` or `hax` invokes `rustc` on the code.
3. `rustc`'s macro expander runs.

4. The `#[requires]` procedural macro runs, rewriting the function:
```rust
#[cfg_attr(hax, ::hax_lib::contract::requires(x > 0))]
fn my_func(x: u32) {
    #[cfg(hax)]
    ::hax_lib::_internal_precondition(::hax_lib::Prop::from(x > 0));

    assert!(x != 0); // This is still a macro
}
```

5. The `assert!` declarative macro runs, rewriting the `assert!`:
```rust
#[cfg_attr(hax, ::hax_lib::contract::requires(x > 0))]
fn my_func(x: u32) {
    #[cfg(hax)]
    ::hax_lib::_internal_precondition(::hax_lib::Prop::from(x > 0));

    // Final expansion depends on `cfg(hax)`

    // If `cfg(hax)` is set:
    #[cfg(hax)]
    ::hax_lib::assert(x != 0);

    // If `cfg(not(hax))` is set:
    #[cfg(not(hax))]
    ::core::assert!(x != 0);
}
```

6. This final, expanded code is what `hax`'s extractor binary (`cargo-hax`) receives as input. It parses this AST, finds the `#[cfg_attr(hax, ...)]` attributes and special marker functions, and translates them into a `.fst` or `.lean` file.

## Complete Syntax Reference
| Macro | Type | Category | Purpose |
| --- | --- | --- | --- |
| `assert!(...)` | Declarative | Assertions | Prove a runtime-computable fact. |
| `assume!(...)` | Declarative | Assertions | Assume a runtime-computable fact. **Unsafe.** |
| `assert_prop!(...)` | Declarative | Assertions | Prove a pure logical (non-computable) fact. |
| `debug_assert!(...)` | Declarative | Assertions | Ignored by `hax`. Runtime check for `debug` builds. |
| `#[requires(...)]` | Procedural Attr. | Contracts | Add a precondition to a function. |
| `#[ensures(...)]` | Procedural Attr. | Contracts | Add a postcondition to a function. |
| `#[lemma]` | Procedural Attr. | Contracts | Define a proof-only "ghost" function. |
| `#[loop_invariant(...)]` | Procedural Attr. | Loops | Add an invariant to a `for`, `while`, or `loop`. |
| `#[decreases(...)]` | Procedural Attr. | Loops | Add a termination measure to a recursive function. |
| `#[loop_decreases(...)]` | Procedural Attr. | Loops | Add a termination measure to a loop. |
| `#[refinement_type]` | Procedural Attr. | Refinements | Define a struct as a refinement type. |
| `#[refinement]` | Procedural Attr. | Refinements | Mark the base-type field in a `#[refinement_type]`. |
| `#[include]` | Procedural Attr. | Visibility | Force-include an item in extraction. |
| `#[exclude]` | Procedural Attr. | Visibility | Force-exclude an item from extraction. |
| `#[opaque]` | Procedural Attr. | Visibility | Hide a function's body from the verifier. |
| `#[opaque_type]` | Procedural Attr. | Visibility | Hide a type's definition from the verifier. |
| `fstar!(...)` | Declarative | Backend | Inject raw F* code. |
| `#[fstar::options(...)]` | Procedural Attr. | Backend | Set F* options for an item. |
| `#[protocol_messages]` | Procedural Attr. | Protocol | Define a protocol message enum. |
| `#[state_machine(...)]` | Procedural Attr. | Protocol | Define and verify a state machine `impl`. |
| `forall!(...)` | Declarative | Mathematical | Logical "for all" quantifier (`∀`). |
| `exists!(...)` | Declarative | Mathematical | Logical "exists" quantifier (`∃`). |
| `implies!(...)` | Declarative | Mathematical | Logical implication (`A ==> B`). |
