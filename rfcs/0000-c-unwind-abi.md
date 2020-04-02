- Feature Name: `"C unwind" ABI`
- Start Date: 2019-04-03
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

# Motivation
[motivation]: #motivation

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

| ABI          | panic runtime  | `panic`-unwind                        | Forced unwind, no destructors | Forced unwind with destructors | Other foreign unwind |
| ------------ | -------------- | ------------------------------------- | ----------------------------- | ------------------------------ | -------------------- |
| `"C"`        | `panic=unwind` | abort                                 | unwind                        | UB                             | UB                   |
| `"C unwind"` | `panic=unwind` | unwind                                | unwind                        | unwind                         | unwind               |
| `"C"`        | `panic=abort`  | `panic!` aborts (no unwinding occurs) | unwind                        | UB                             | UB                   |
| `"C unwind"` | `panic=abort`  | `panic!` aborts                       | unwind                        | UB                             | abort                |

# Drawbacks
[drawbacks]: #drawbacks

Still some UB

(more: see notes from design meeting)

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Let foreign exceptions cross `extern "C"` boundary ("option 3" in blog post)

Variant on this approach - "option 1" in blog post: different "forced-unwind"
behavior

# Prior art
[prior-art]: #prior-art

C++

# Unresolved questions
[unresolved-questions]: #unresolved-questions



# Future possibilities
[future-possibilities]: #future-possibilities
