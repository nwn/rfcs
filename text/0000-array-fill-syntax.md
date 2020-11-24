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

It can be useful to insert an existing array into an array being created. This can be done using _array expansions_.

Any of the comma-separated elements in an array expression may be replaced with an array expansion of the form `..<expr>`, where `<expr>` is itself an array. This inserts each element of `<expr>` into the new array at that position as though they had been written out explicitly. The array being defined and `<expr>` must therefore have equivalent element types.

For example, say we wish to define `array` as follows:
``` Rust
let sub_array = [3, 4];
let array = [1, 2, sub_array[0], sub_array[1], 5, 6];
assert_eq!(array, [1, 2, 3, 4, 5, 6]);
```

This could be written more concisely using an array expansion:
``` Rust
let sub_array = [3, 4];
let array = [1, 2, ..sub_array, 5, 6];
assert_eq!(array, [1, 2, 3, 4, 5, 6]);
```

This notation is even necessary when inserting an array of `!Copy` elements. For example, the following snippet does not work since the value of `sub_array` is moved by `sub_array[0]` before we can move `sub_array[1]`:
``` Rust
let sub_array = [String::from("1"), String::from("2")];
let array = [String::from("0"), sub_array[0], sub_array[1], String::from("3")];
```

Instead we can move the entirety of `sub_array` at once:
``` Rust
let sub_array = [String::from("1"), String::from("2")];
let array = [String::from("0"), ..sub_array, String::from("3")];
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This proposal does not affect existing array expressions: array expressions without an array expansion and repeat-style array expressions still behave exactly the same. This only extends array expressions.

### Desugaring

The examples in the guide-level explanation can be considered to desugar as follows:
``` Rust
let sub_array = [3, 4];
let array = [1, 2, ..sub_array, 5, 6]; // This desugars to the expression below.

let array = {
    let elem_0 = 1;
    let elem_1 = 2;
    let [elem_2, elem_3] = sub_array;
    let elem_4 = 5;
    let elem_5 = 6;
    [elem_0, elem_1, elem_2, elem_3, elem_4, elem_5]
};
```

``` Rust
let sub_array = [String::from("1"), String::from("2")];
let array = [String::from("0"), ..sub_array, String::from("3")]; // This desugars to the expression below.

let array = {
    let elem_0 = String::from("0");
    let [elem_1, elem_2] = sub_array;
    let elem_3 = String::from("3");
    [elem_0, elem_1, elem_2, elem_3]
};
```

Note that the array expansion is evaluated exactly once, even if it expands to no elements. This matches the behaviour of repeat-style arrays of length 0. An array containing only an array expansion behaves exactly like a copy/move of the expanded expression. This means the following are also equivalent:
``` Rust
let x = [ ..some_array() ];
let x = some_array();
```
``` Rust
let x = [ ..[{ println!("Side effects"); true }; 0] ];
let x = [ { println!("Side effects"); true }; 0 ];
let x = {
    let _elem = { println!("Side effects"); true };
    []
};
```

### Length Inference

The length of an array expression with expansions is the sum of:

- The number of non-expansion elements in the expression
- The length of each array being expanded

### Errors
Several errors can arise when using this feature. Each of these is an instance of existing errors.

- If the computed length of an array expression does not match the expected length, this is treated as a type mismatch, as usual:
  ``` Rust
  let x: [u32; 3] = [..[42; 2]];
  ```
  the following error is produced:
  ``` Rust
  error[E0308]: mismatched types
   --> src/main.rs:2:23
    |
  2 |     let x: [u32; 3] = [..[42; 2]];
    |            --------   ^^^^^^^^^^^ expected an array with a fixed size of 3 elements, found one with 2 elements
    |            |
    |            expected due to this
  ```

- If array expansions are used to create a circular definition, the usual error results:
  ``` Rust
  const X: [u32; 1] = [..Y];
  const Y: [u32; 1] = [..X];
  ```
  The following error is produced:
  ``` Rust
  error[E0391]: cycle detected when simplifying constant for the type system `X`
   --> src/main.rs:2:1
    |
  2 | const X: [u32; 1] = [..Y];
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^
    |
  note: ...which requires simplifying constant for the type system `Y`...
   --> src/main.rs:3:1
    |
  3 | const Y: [u32; 1] = [..X];
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^
    = note: ...which again requires simplifying constant for the type system `X`, completing the cycle
  ```

- If the element type of an expanded array does not match that of the other elements:
  ``` Rust
  let x = [true, false, ..[7]];
  ```
  yields the following error:
  ``` Rust
  error[E0308]: mismatched types
   --> src/main.rs:2:19
    |
  2 |     let x = [true, false, ..[7]];
    |                              ^ expected `bool`, found integer
    = note: expected array `[bool; _]`
               found array `[u32; 1]`
  ```

- If the expression in an array expansion is not an array:
  ``` Rust
  let x = [true, false, ..7];
  ```
  yields the following error:
  ``` Rust
  error[E0308]: mismatched types
   --> src/main.rs:2:19
    |
  2 |     let x = [true, false, ..7];
    |                             ^ expected array `[bool; _]`, found integer
  ```

- If the element type of an array cannot be determined:
  ``` Rust
  let x = [..[]];
  ```
  yields the following error:
  ``` Rust
  error[E0282]: type annotations needed for `[_; 0]`
  --> src/main.rs:2:13
    |
  2 |     let x = [..[]];
    |         -      ^^ cannot infer type
    |         |
    |         consider giving `x` the explicit type `[_; 0]`, with the type parameters specified
  ```

### Interactions with `RangeTo`
[interactions-with-rangeto]: #interactions-with-rangeto

The `..<expr>` syntax is given special meaning within an array expression with higher precedence than the `RangeTo` operator. This only applies when the _array expansion_ is a direct child of the array expression. Other range operators are unaffected. This means that the following hold:
``` Rust
let x = [1, 2, 3, ..[0; 3]]; // Interpreted as an array expansion, not a range expression
assert_eq!(x, [1, 2, 3, 0, 0, 0]);

let x = [..[2]]; // Interpreted as an array expansion, not a range expression
assert_eq!(x, [2]);

// Other range expressions are unaffected
assert_eq!([0..2], [Range { start: 0, end: 2 }]);
assert_eq!([..], [RangeFull]);
assert_eq!([2..], [RangeFrom { start: 2 }]);
assert_eq!([0..=2], [RangeInclusive::new(0, 2)]);
assert_eq!([..=2], [RangeToInclusive { end: 2 }]);
```

When not a direct child of the array expression, the `RangeTo` operator is unaffected. That is, subexpressions and parenthesized expressions are unaffected. Therefore the following also hold:
``` Rust
// Parentheses prevent the expression from being treated as an array expansion.
assert_eq!([(..2)], [RangeTo { end: 2 }]);

// Subexpressions of an element are also treated normally.
let x: [RangeTo<u32>; 1] = [{ ..2 }]; // `RangeTo` expression within a block expression
let x: [RangeTo<u32>; 2] = [.. ..2]; // `RangeTo` expression within an array expansion (parses, but yields a type error)
```

# Drawbacks
[drawbacks]: #drawbacks

### Breakage

The biggest drawback is that this is a breaking change to the language. As discussed [above](#interactions-with-rangeto), existing code using arrays of `RangeTo` literals would conflict with this syntax and fail to compile. The exact interactions are detailed above.

The breakage introduced can be handled relatively harmlessly.

- To our knowledge, broken code (`RangeTo` literals within array expressions) is used extremely rarely. A crater run should be performed to confirm this.
- Such breakages can be easily detected and programmatically fixed across an edition boundary.
  - `RangeTo` literals can be parenthesized to avoid the conflict.
  - The resulting type mismatch can be easily recognized.
- Code with such conflicts will almost always fail to compile, rather than miscompile. Miscompilation only occurs when:
  - The array only contains literal `RangeTo` expressions on array types, and
  - The array's type is not specified in any of its other use.

This breakage can also be considered to extend to users' mental models. Users may be confused when their `RangeTo` expression unexpectedly yields a type error about array expansions. This can be mitigated by detecting when such errors occur in a `[RangeTo<_>; _]` literal, and suggesting that the element be parenthesized.
``` Rust
let first = ..4;
let array = [first, ..5];
```
``` Rust
error[E0308]: mismatched types
 --> src/main.rs:2:26
  |
2 |     let array = [first, ..5];
  |                           ^ expected array of `u32`, found integer
help: try using parentheses here:
  |
2 |     let array = [first, (..5)];
  |                         ^^^^^
```

### Further overloading `..`

The proposed syntax adds yet another meaning to the `..` symbol, which some already consider too overloaded. Our hope here is to align with users' existing familiarity with `..` in struct update contexts, making this "expanding on a meaning" rather than "adding a meaning".

### Limitations

This syntax does not work in pattern positions, where it conflicts with unstable half-open range patterns. However, precedence for such differences between expressions and patterns exists. Both `RangeTo` literals and repeat-style array literals (e.g. `[true; 5]`) cannot appear in patterns, so it is not too surprising that array expansions (as proposed here) also cannot.

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
- Could this be implemented as a macro without loss of usability?
- Are array expansions "transparent" to eventual length inference?

# Future possibilities
[future-possibilities]: #future-possibilities

- This syntax can be implemented for dynamically-sized slices in the `vec!` macro.
- The proposed syntax does not preclude any of the existing indexed array initializer proposals.
