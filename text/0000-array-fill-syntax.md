- Feature Name: `array_expansion_syntax`
- Start Date: 2020-03-26
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow shorthand syntax for inserting a subarray into an array literal. For example:
``` Rust
let x = [3, 2, 1];
let y = [4, ..x, 0]; // Insert the elements of `x` into the literal
assert_eq!(y, [4, 3, 2, 1, 0]);
```

# Motivation
[motivation]: #motivation

### Simpler array definitions

Concatenation is a fundamental operation of arrays. Yet in current Rust, there's no way to express array concatenation at compile time. Even with the eventual stabilization of `const` library features like `copy_from_slice`, the user must specify the length of the resulting array, which is not necessarily known to them (though it is always to the compiler).

For example, consider the concatenation of two run-time evaluated arrays. Even leaving aside the actual concatenation, the length of the resulting array cannot be determined by the user at compile time.
``` Rust
let a = compute_a();
let b = compute_b();
let c: [u8; a.len() + b.len()] = concat_arrays(a, b);
//          ~~~~~~~~~~~~~~~~~
//           Error: attempt to use a non-constant value in a constant
```
Since the contents of `a` and `b` are not known at compile time, we are unable to obtain their combined length, though this is known to the compiler. This proposal would allow the user to leverage the compiler to sidestep this issue entirely.
``` Rust
let a = compute_a();
let b = compute_b();
let c = [..a, ..b];
```
This syntax aims to provide an intuitive and generalized notation for building arrays from their constituents.

### The RLE pattern

In C and C++, it is common to define arrays by their first few elements, letting the remainder be default-initialized, like so:
``` C++
int int_array[6] = {1, 2, 3}; // Final elements initialized to 0
array<string_view, 6> str_array = {"Hello", "World"}; // Final elements initialized to ""
```
Rust currently has no analogous syntax. With this proposal, this gap could be filled like so:
``` Rust
let int_array = [1, 2, 3, ..[0; 3]];
let str_array = ["Hello", "World", ..[""; 4]];
```

This syntax is in fact more powerful than the C/C++ version in that arbitrary values can be inserted into the array, rather than just repeating the default-initialized value.
``` Rust
let alt = [Some(1.0), None];
let seq = [..alt, ..alt, ..alt];
```

Moreover, it allows multiple expansions to occur anywhere within an array literal. This effectively allows a run-length encoding of array literals:
``` Rust
let rle = [1, ..[0; 32], 2, 3, 4, ..[-1; 28]];
```

``` Rust
let zimin0 = [0];
let zimin1 = [..zimin0, 1, ..zimin0];
let zimin2 = [..zimin1, 2, ..zimin1];
let zimin3 = [..zimin2, 3, ..zimin2];
assert_eq!(zimin3, [0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0]);
```

Either of the above examples using the existing syntax would require care both when transcribing and when reading for patterns.

### Less redundancy

Repetition in code can lead to bugs and reduce the readability of the code. A large array with many repeated elements is not currently obvious with the existing syntax. A reader of the code would be required to scan the entire array to determine any deviance.

Similar problems occur when defining subarrays, where separately defined literals can be less readable and can easily fall out of sync.
``` Rust
const PNG_HEADER: [u8; 8] = [ 0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a ];
const PNG_IHDR: [u8; 4] = [ 0x49, 0x48, 0x44, 0x52 ];
const PNG_IDAT: [u8; 4] = [ 0x49, 0x44, 0x41, 0x54 ];
const PNG_IEND: [u8; 4] = [ 0x49, 0x45, 0x4e, 0x44 ];
const IMAGE: [u8; 76] = [
    ..PNG_HEADER,
    0x00, 0x00, 0x00, 0x0d, ..PNG_IHDR,
    0x00, 0x00, 0x00, 0x01, 0x00, 0x00, 0x00, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00,
    0x37, 0x6e, 0xf9, 0x24,
    0x00, 0x00, 0x00, 0x10, ..PNG_IDAT,
    0x78, 0x9c, 0x62, 0x60, 0x01, 0x00, 0x00, 0x00, 0xff, 0xff, 0x03, 0x00, 0x00, 0x06, 0x00, 0x05,
    0x57, 0xbf, 0xab, 0xd4,
    0x00, 0x00, 0x00, 0x00, ..PNG_IEND,
    0xae, 0x42, 0x60, 0x82,
];
```

The proposed syntax makes such code more convenient to write and more clear to read.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

An array expression can be written by enclosing zero or more comma-separated expressions of uniform type in square brackets. This produces an array containing each of these values in the order they are written. Such an array expression can optionally be terminated with an element of the form `..<expr>`, where `<expr>` is of the same type as the preceeding elements. This appends zero or more copies of `expr` (the _fill expression_) to the produced array. The length of the array is at least the number of elements before the fill expression, and is determined exactly by type inference.

If the fill expression fills more than one element, then the element type must be `Copy`. The fill expression is evaluated exactly once and moved or copied to the necessary number of elements.

For example:
``` Rust
let x: [i32; 6] = [1, 2, 3, ..0]; // The last 3 elements are 0
assert_eq!(x, [1, 2, 3, 0, 0, 0]);
```

This becomes useful when dealing with many repeated elements or when the elements are complex:
``` Rust
let x: [(Option<f32>, usize, bool); 20] = [
    (Some(1.0), 0x42, true),
    (None, 0x1a, false),
    ..(Some(0.4), 0xfe, true) // This term is repeated 18 times
];
```

It is possible for the fill expression to not fill any elements in the array:
``` Rust
let x: [char; 5] = ['H', 'e', 'l', 'l', 'o', ..'\0']; // All elements are already specified
assert_eq!(x, ['H', 'e', 'l', 'l', 'o']);
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This proposal does not affect the other types of array expressions: fully expanded array expressions without a fill expression  and repeat-style array expressions still behave exactly the same. This instead introduces a third type of array expression.

### Desugaring

The examples in the guide-level explanation can be considered to desugar as follows:
``` Rust
let x: [i32; 6] = [1, 2, 3, ..0]; // This desugars to the expression below.

let x: [i32; 6] = {
    let elem_0 = 1;
    let elem_1 = 2;
    let elem_2 = 3;
    let elem_fill = 0;
    [elem_0, elem_1, elem_2, elem_fill, elem_fill, elem_fill]
};
```

``` Rust
let x: [(Option<f32>, usize, bool); 20] = [
    (Some(1.0), 0x42, true),
    (None, 0x1a, false),
    ..(Some(0.4), 0xfe, true)
]; // This desugars to the expression below.

let x: [(Option<f32>, usize, bool); 20] = {
    let elem_0 = (Some(1.0), 0x42, true);
    let elem_1 = (None, 0x1a, false);
    let elem_fill = (Some(0.4), 0xfe, true);
    [elem_0, elem_1, elem_fill, elem_fill, elem_fill,
     elem_fill, elem_fill, elem_fill, elem_fill, elem_fill,
     elem_fill, elem_fill, elem_fill, elem_fill, elem_fill,
     elem_fill, elem_fill, elem_fill, elem_fill, elem_fill]
};
```

``` Rust
let x: [char; 5] = ['H', 'e', 'l', 'l', 'o', ..'\0']; // This desugars to the expression below.

let x: [char; 5] = {
    let elem_0 = 'H';
    let elem_1 = 'e';
    let elem_2 = 'l';
    let elem_3 = 'l';
    let elem_4 = 'o';
    let _elem_fill = '\0'; // The fill expression still gets evaluated
    [elem_0, elem_1, elem_2, elem_3, elem_4]
};
```

Note that the fill expression is evaluated exactly once, even if it fills no elements. This matches the behaviour of repeat-style arrays of length 0. An array containing only a fill expression behaves exactly like a repeat-style array expression of the same length. This means the following are also equivalent:
``` Rust
let x: [bool; 0] = [ ..{ println!("Side effects"); true } ];
let x: [bool; 0] = [ { println!("Side effects"); true }; 0 ];
let x: [bool; 0] = {
    let _elem_fill = { println!("Side effects"); true };
    []
};
```

### Length Inference

The length of an array expression with a fill expression is determined by type inference:

- If an exact length can be _uniquely_ determined from the surrounding program context, the array expression has that length.
- If the program context under-constrains or over-constrains the length, it is considered a static type error.

So this (in isolation) is a type error:
``` Rust
let x = [..true]; // Length is under-constrained
```
This is also a type error:
``` Rust
let x = [..true]; // Length is determined from uses of `x`
let y: [bool; 3] = x; // Fixes length of `x` to 3
let z: [bool; 4] = x; // Error: array length mismatch
```
But this is valid:
``` Rust
let x: [[bool; 4]; 2] = [[..true], [true, ..false]]; // Each sub-array has length 4
```

#### Errors
Several errors can arise when using this feature.

- If the length is under-constrained as in the following code:
  ``` Rust
  let x = [..true];
  ```
  the following error is produced:
  ``` Rust
  error[E0282]: type annotations needed
   --> src/main.rs:4:9
    |
  4 |     let x = [..true];
    |         ^   ^^^^^^^^
    |         |   |
    |         |   cannot infer length for array
    |         help: consider giving `x` a type
  ```
  
- If the length has not yet been fixed, a type mismatch yields a slightly different error as follows:
  ``` Rust
  let x: [bool; 3] = [true, false, true, false, ..true];
  ```
  yields:
  ``` Rust
  error[E0308]: mismatched types
   --> src/main.rs:4:24
    |
  4 |     let x: [bool; 3] = [true, false, true, false, ..true];
    |            ---------   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected an array with a fixed size of 3 elements, found one with at least 4 elements
    |            |
    |            expected due to this
  ```
  
- Once a length has been assigned to a filled array expression, array length errors act as normal. The following code:
  ``` Rust
  let x = [..true];
  let y: [bool; 3] = x;
  let z: [bool; 4] = x;
  ```
  yields the following error:
  ``` Rust
  error[E0308]: mismatched types
    --> src/main.rs:10:22
     |
  10 |     let z: [bool; 4] = x;
     |            ---------   ^ expected an array with a fixed size of 4 elements, found one with 3 elements
     |            |
     |            expected due to this
  ```

### Interactions with `RangeTo`
[interactions-with-rangeto]: #interactions-with-rangeto

The `..<expr>` syntax is given special meaning within an array expression with higher precedence than the `RangeTo` operator. This only applies when the _fill expression_ is a direct child of the array expression. Other range operators are unaffected. This means that the following hold:
``` Rust
let x: [u32; 6] = [1, 2, 3, ..0]; // Interpreted as a fill expression, not a range expression
assert_eq!(x, [1, 2, 3, 0, 0, 0]);

let x: [u32; 3] = [..2]; // Interpreted as a fill expression, not a range expression
assert_eq!(x, [2, 2, 2]);

// Other range expressions are unaffected
assert_eq!([0..2], [Range { start: 0, end: 2 }]);
assert_eq!([..], [RangeFull]);
assert_eq!([2..], [RangeFrom { start: 2 }]);
assert_eq!([0..=2], [RangeInclusive::new(0, 2)]);
assert_eq!([..=2], [RangeToInclusive { end: 2 }]);
```

When not a direct child of the array expression, the `RangeTo` operator is unaffected. That is, subexpressions and parenthesized expressions are unaffected. Therefore the following also hold:
``` Rust
// Parentheses prevent the expression from being treated as a fill expression.
assert_eq!([(..2)], [RangeTo { end: 2 }]);

// Subexpressions of an element are also treated normally.
let x: [RangeTo<u32>; 2] = [.. ..2]; // `RangeTo` expression within a fill expression
assert_eq!(x, [RangeTo { end: 2 }, RangeTo { end: 2 }]);
assert_eq!([{ ..2 }], [RangeTo { end: 2 }]); // `RangeTo` expression within a brace expression
```

# Drawbacks
[drawbacks]: #drawbacks

### Breakage

The biggest drawback is that this is a breaking change to the language. As discussed [above](#interactions-with-rangeto), existing code using arrays of `RangeTo` literals would conflict with this syntax and fail to compile. To our knowledge, such code is used extremely rarely and can easily be fixed. The exact interactions are detailed above.

### Limitations

This syntax does not work in pattern positions, where it conflicts with unstable half-open range patterns. However, precedence for such differences between expressions and patterns exists. Both `RangeTo` literals and repeat-style array literals (e.g. `[true; 5]`) cannot appear in patterns, so it is not too surprising that fill-style array literals (as proposed here) also cannot.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Rationale

The proposed syntax was chosen mainly for its familiarity. It reflects the meaning of the `..` in the struct update syntax, namely "copy/move fields/elements from what follows". It should also be intuitive for newcomers from Javascript since the `..` acts similarly to the `...` in Javascript's spread syntax.

### Alternatives
[alternatives]: #alternatives

- Implementing this as a macro in either std or an external crate. This may not currently be possible due to the lengths of non-`const` arrays not being available in `const` contexts. ([For example](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=4d8bc434e45aa2103ab874b873359dc4))

- An alternative syntax that doesn't conflict with `RangeTo`, e.g. `assert_eq!([1, 2, 3, 3, 3], [1, 2, ...[3; 3]])` leveraging the existing (but unused) `...` token.

# Prior art
[prior-art]: #prior-art

### Javascript
Javascript has a similar, though less restrictive, feature in its [spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax), which allows arbitrary iterables to be expanded to discrete arguments to a function call or elements to an array literal. The spread syntax also works for object expressions, where it acts similarly to Rust's `struct` update (FRU) syntax.

This proposal can be thought to similarly extend Rust's existing "spread" syntax to the context of array literals.

### C and C++
As described [above](#motivation), a related feature is present in C and C++. In C, missing elements in an initializer are implicitly initialized to zero (NULL, etc.). C++ improves on this design by default-initializing any missing elements, allowing for more complex types in this position.

Both C and C++ suffer from the problem that arrays are _silently_ and _implicitly_ filled when elements are missing. This can lead to unexpected behaviour and bugs. Still, the convenience of this feature means that it continues to be used frequently. The proposed feature solves this problem while improving usability by making the behaviour explicit and opt-in, and by using only user-defined values.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is this the best syntax for such a feature?
- Could this be implemented as a macro?

# Future possibilities
[future-possibilities]: #future-possibilities

- This syntax can be implemented for dynamically-sized slices in the `vec!` macro.
- The proposed syntax does not preclude any of the existing indexed array initializer proposals.
