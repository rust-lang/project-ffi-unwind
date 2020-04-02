- Feature Name: `"C unwind" ABI`
- Start Date: 2019-04-03
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Introduces a new ABI string, `"C unwind"`, to enable unwinding from other
languages (such as C++) into Rust frames and from Rust into other languages.

Additionally, defines the behavior for a limited number of previously-undefined
cases when an unwind operation reaches a Rust function boundary with a
non-`"Rust"`, non-`"C"` ABI.

# Motivation
[motivation]: #motivation

There are some Rust projects that need cross-language unwinding to provide
their desired functionality. One major example is WASM interpreters, including
the Lucet and Wasmer projects.

There are also existing Rust crates (notably, wrappers around the `libpng` and
`libjpeg` C libraries) that rely on compatibility with the native exception
handling mechanism in GCC, LLVM, and MSVC.

Additionally, there are libraries such as `rlua` that rely on `longjmp` across
Rust frames; on Windows, `longjmp` is implemented via unwinding.

<!-- TODO: links to prior discussions? -->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When declaring an external function that may unwind, such as an entrypoint to a
C++ library or a `libc` function that may invoke `pthread_exit`, use
`"C unwind"`:

```
extern "C unwind" {
  fn may_throw();
}
```

Rust functions that call a possibly-unwinding external function should either
use the default Rust ABI (which can be made explicit with `extern "Rust"`) or
the `"C unwind"` ABI:

```
extern "C unwind" fn can_unwind() {
  // ...
```

Using the `"C unwind"` ABI to "sandwich" Rust frames between frames from
another language (such as C++) allows an exception initiated in a callee frame
in the other language to traverse the intermediate Rust frames before being
caught in the caller frames. I.e., a C++ exception may be thrown,
cross into Rust via an `extern "C unwind"` function declaration, safely unwind
the Rust frames, and cross back into C++ (where it may be caught) via a Rust
`"C unwind"` function definition.

Conversely, languages that support the native unwinding mechanism, such as C++,
may be "sandwiched" between Rust frames, so that Rust `panic`s may safely
unwind the C++ frames, if the Rust code declares both the C++ entrypoint and
the Rust entrypoint using `"C unwind"`.

## Changes to `extern "C"` behavior

Prior to this RFC, any unwinding operation that crossed an `extern "C"`
boundary, either from a `panic!` "escaping" from a Rust function defined with
`extern "C"` or by entering Rust from another language via an entrypoint
declared with `extern "C"`, caused undefined behavior.

This RFC retains most of that undefined behavior, with a few exceptions:

<!-- TODO -->

## Interaction with `panic=abort`

When compiling with the `panic=abort` panic runtime, there are some types of
unwinding that are not guaranteed cause the program to abort:

* Forced unwinding: this is a special kind of unwinding used to implement
  `longjmp` on Windows and `pthread_exit` in `glibc`;
  <!-- TODO: link to blog post, or is this sufficient? -->
  Rust provides no mechanism to catch this type of unwinding.
* Unwinding from another language into Rust if the entrypoint to that language
  is declared with `extern "C"` (contrary to the guidelines above)

In the case of forced unwinding, this is safe as long as none of the unwound Rust frames contain destructors. In the case of an `extern "C"` function

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

| panic runtime  | ABI          | `panic`-unwind                        | Forced unwind, no destructors | Forced unwind with destructors | Other foreign unwind |
| -------------- | ------------ | ------------------------------------- | ----------------------------- | ------------------------------ | -------------------- |
| `panic=unwind` | `"C unwind"` | unwind                                | unwind                        | unwind                         | unwind               |
| `panic=unwind` | `"C"`-like   | abort                                 | unwind                        | UB                             | UB                   |
| `panic=abort`  | `"C unwind"` | `panic!` aborts                       | unwind                        | UB                             | abort                |
| `panic=abort`  | `"C"`-like   | `panic!` aborts (no unwinding occurs) | unwind                        | UB                             | UB                   |

<!-- TODO: note about `catch_unwind` -->

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

