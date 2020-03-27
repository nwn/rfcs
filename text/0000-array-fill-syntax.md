- Feature Name: `array-fill-syntax`
- Start Date: 2020-03-26
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

---

Allow shorthand syntax for array literals with the same element repeated at the end. For example:
``` Rust
let x: [i32; 6] = [1, 2, 3, ..0]; // The last 3 elements are 0
assert_eq!(x, [1, 2, 3, 0, 0, 0]);
```

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

---

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

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

---

- Breaks arrays of RangeTo (`[..1, ..2, ..3]`)
- Doesn't work in pattern position (unexpectedly due to unstable half-open range patterns; RangeTo doesn't work unparenthesized in patterns)
- Only affects a single run of identical elements at the end of an array (can't do `[1, ..2, ..3]`)

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

---

- Reflective of the update-syntax in struct expressions.
- Currently not possible with `macro_rules!` since "repeat N times" is not expressible. Possibly with macros 2.0?
- A more general syntax for run-length encoding array literals. This would solve the earlier drawback of multiple runs. However, in the real world, most cases involving multiple runs would require sufficient granularity that such a feature would provide little benefit.

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

---

As described [above](#motivation), a similar syntax is present in C and C++. In C, missing elements in an initializer are implicitly initialized to zero (NULL, etc.). C++ improves on this design by default-initializing any missing elements, allowing for more complex types in this position.

Both C and C++ suffer from the problem that arrays are _silently_ and _implicitly_ filled when elements are missing. This can lead to unexpected behaviour and bugs. Still, the convenience of this feature means that it continues to be used frequently. The proposed feature solves this problem while improving usability by making the behaviour explicit and opt-in, and by using only a user-defined value.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

---

- Should we warn/err when a fill expression would expand to 0 entries? Clippy?
- Should this allow `!Copy` types if the expression does not expand to more than one entry? This aligns with the `[vec![]; 1]` syntax.

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

---

- This could be extended to allow middle-filling: `[1, 2, ..3, 2, 1]`
