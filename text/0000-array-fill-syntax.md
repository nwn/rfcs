- Feature Name: `array-fill-syntax`
- Start Date: 2020-03-26
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow shorthand syntax for array literals with the same element repeated at the end. For example:
``` Rust
let x: [i32; 6] = [1, 2, 3, ..0]; // The last 3 elements are 0
assert_eq!(x, [1, 2, 3, 0, 0, 0]);
```

# Motivation
[motivation]: #motivation

### Parity with Other Languages

In C and C++, it is common to define arrays by their first few elements, letting the remainder be default-initialized, like so:
``` C++
int int_array[6] = {1, 2, 3}; // Final elements initialized to 0
array<string_view, 6> str_array = {"Hello", "World"}; // Final elements initialized to ""
```
Rust currently has no analogous syntax. We propose to fill this gap with the following syntax:
``` Rust
let int_array: [i32; 6] = [1, 2, 3, ..0];
let str_array: [&str; 6] = ["Hello", "World", ..""];
```

This syntax is in fact more powerful than the C/C++ version in that it supports arbitrary `Copy` values to be filled into the array:
``` Rust
let mostly_some: [Option<f32>; 100] = [Some(0.0), Some(1.0), None, ..Some(-1.0)];
```

### Code readability

Repetition in code can both be a source of bugs and reduce the readability of the code. A large array wherein most of the elements are identical is not currently obvious with the existing syntax. A reader of the code would be required to scan the entire array to determine any deviance.

The proposed syntax makes this both more convenient when writing such code and more clear when reading such code.

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

The biggest drawback is that this is a breaking change. As discussed [above](#interactions-with-rangeto), existing code using arrays of `RangeTo` literals would conflict with this syntax and fail to compile. To our knowledge, such code is used extremely rarely and can easily be fixed. The exact interactions are detailed above.

### Inferred Lengths

The proposed syntax hides the actual length of an array literal. Unlike the two existing array forms (`[1, 2, 3]` and `[true; 5]`), the length of the array cannot be determined from the expression alone. This can hinder readability. However, in use cases where one would prefer the fill-syntax, the only alternative is a fully expanded array of sufficient length that this information is effectively hidden from the reader anyway. In such cases, explicit type annotations can be used.

This would also complicate the compiler's job of type inference.

### Limitations

This syntax does not work in pattern positions, where it conflicts with unstable half-open range patterns. However, precedence for such differences between expressions and patterns exists. Both `RangeTo` literals and repeat-style array literals (e.g. `[true; 5]`) cannot appear in patterns, so it is reasonable to expect that fill-style array literals (as proposed here) also cannot.

This syntax does not afford extensions to arbitrary run-length encoded arrays, as described in the [alternatives](#alternatives).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Rationale

The proposed syntax was chosen mainly for its familiarity. It is more intuitive for newcomers since the `..` acts similarly to the ellipsis in both English and mathematical notation. It also reflects the meaning of the `..` in the struct update syntax, namely "copy/move the remaining fields/elements from what follows".

### Alternatives
[alternatives]: #alternatives

- Implementing this as a macro in either std or an external crate. Not sure if this is actually possible for compile-time evaluation without `const` loops.

- An alternative syntax:
  - Extend the repeat-syntax instead of the expanded syntax. This makes the length explicit, but the syntax would be less intuitive and noticeable: \
    `assert_eq!([1, 2, 3, 3, 3], [1, 2, 3; 5])` or \
    `assert_eq!([1, 2, 3, 3, 3], [1, 2, ..3; 5])`
  - Use a syntax that doesn't conflict with `RangeTo`, e.g. `assert_eq!([1, 2, 3, 3, 3], [1, 2, 3...])` leveraging the existing (but unused) `...` token

- A more general syntax for run-length encoding array literals. This would solve the earlier drawback of multiple runs. However, in the real world, most cases involving multiple runs would require sufficient granularity that such a feature would provide little benefit.

# Prior art
[prior-art]: #prior-art

As described [above](#motivation), a similar feature is present in C and C++. In C, missing elements in an initializer are implicitly initialized to zero (NULL, etc.). C++ improves on this design by default-initializing any missing elements, allowing for more complex types in this position.

Both C and C++ suffer from the problem that arrays are _silently_ and _implicitly_ filled when elements are missing. This can lead to unexpected behaviour and bugs. Still, the convenience of this feature means that it continues to be used frequently. The proposed feature solves this problem while improving usability by making the behaviour explicit and opt-in, and by using only a user-defined value.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is this the best syntax for such a feature?
- Should we (or Clippy) warn when a fill expression would expand to 0 entries? \
  Allowing this could have valid use cases and aligns with the lack of warning when a base struct contributes no fields.
- Should this allow `!Copy` types if the expression does not expand to more than one entry? \
  This aligns with the `[vec![]; 1]` syntax.

# Future possibilities
[future-possibilities]: #future-possibilities

- This could be extended to allow middle-filling: `[1, 2, ..3, 2, 1]`
