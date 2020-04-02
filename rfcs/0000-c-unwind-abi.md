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

This RFC does not define the behavior of `catch_unwind` in a Rust frame being
unwound by a foreign exception.

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

## Forced unwinding

This is a special kind of unwinding used to implement `longjmp` on Windows and
`pthread_exit` in `glibc`.
<!-- TODO: link to blog post, or is this sufficient? --> 

## Changes to `extern "C"` behavior

Prior to this RFC, any unwinding operation that crossed an `extern "C"`
boundary, either from a `panic!` "escaping" from a Rust function defined with
`extern "C"` or by entering Rust from another language via an entrypoint
declared with `extern "C"`, caused undefined behavior.

This RFC retains most of that undefined behavior, with two exceptions:

* With the `panic=abort` runtime, `panic!` will cause an `abort` if it would
  otherwise "escape" from a function defined with `extern "C"`.
* Forced unwinding is safe with `extern "C"` as long as no frames with
  destructors (i.e. `Drop` types) are unwound.

## Interaction with `panic=abort`

When compiling with the `panic=abort` panic runtime, there are some types of
unwinding that are not guaranteed cause the program to abort:

* Forced unwinding: Rust provides no mechanism to catch this type of unwinding.
* Unwinding from another language into Rust if the entrypoint to that language
  is declared with `extern "C"` (contrary to the guidelines above)

In the case of forced unwinding, this is safe if the new `"C unwind"` ABI is
used and none of the unwound Rust frames contain destructors. In the case of an
unwind coming from an `extern "C"` function, this behavior is always undefined.

If a non-forced foreign unwind would enter a Rust frame via an `extern "C
unwind"` ABI boundary, but the Rust code is compiled with `panic=abort`, the
unwind will be caught and the process aborted.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This table shows the behavior of an unwinding operation reaching each type of
ABI boundary (function declaration or definition). "UB" stands for undefined
behavior.

| panic runtime  | ABI          | `panic`-unwind                        | Forced unwind, no destructors | Forced unwind with destructors | Other foreign unwind |
| -------------- | ------------ | ------------------------------------- | ----------------------------- | ------------------------------ | -------------------- |
| `panic=unwind` | `"C unwind"` | unwind                                | unwind                        | unwind                         | unwind               |
| `panic=unwind` | `"C"`-like   | abort                                 | unwind                        | UB                             | UB                   |
| `panic=abort`  | `"C unwind"` | `panic!` aborts                       | unwind                        | UB                             | abort                |
| `panic=abort`  | `"C"`-like   | `panic!` aborts (no unwinding occurs) | unwind                        | UB                             | UB                   |

No subtype relationship is defined between functions or function pointers using
different ABIs. This RFC also does not define coercions between `"C"` and
`"C unwind"`.

As noted above, if a Rust frame is unwound by a foreign exception, the behavior
is undefined for now.

# Drawbacks
[drawbacks]: #drawbacks

There is still an instance of undefined behavior with this proposal: when a
forced unwind crosses a Rust frame with destructors, the behavior under
`panic=abort` is undefined. This means that there are some cases in which a
program that is well-defined under `panic=unwind` will not be well-defined
under `panic=abort`.

This design imposes some burden on existing codebases (mentioned
[above][motivation]) to change their `extern` annotations to use the new ABI.

Having separate ABIs for `"C"` and `"C unwind"` may make interface design more
difficult, especially since this RFC [postpones][future-possibilities]
introducing coercions between function types using different ABIs.

<!-- TODO more: see notes from design meeting -->

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- TODO: 'requirements' from inside-rust post -->

## Other proposals discussed with the lang team
[alternatives]: #other-proposals-discussed-with-the-lang-team

Two other potential designs have been discussed in depth; they are
explained in [this blog post][inside-rust-post]. The design in this RFC is
referred to as "option 2" in that post.

"Option 1" in that blog post only differs from the current proposal in the
behavior of a forced unwind across a `"C unwind"` boundary under `panic=abort`.
Under the current proposal, this type of unwind is permitted, allowing
`longjmp` and `pthread_exit` to behave "normally" across all platforms
regardless of panic runtime. If there are destructors in the unwound frames,
this results in undefined behavior. Under "option 1", however, all foreign
unwinding, forced or unforced, is caught at `"C unwind"` boundaries under
`panic=abort`, and the process is aborted. This gives `longjmp` and
`pthread_exit` surprising behavior on some platforms, but avoids that cause of
undefined behavior in the current proposal.

The other proposal in the blog post, "option 3", is dramatically different. In
that proposal, foreign exceptions are permitted to cross `extern "C"`
boundaries, and no new ABI is introduced.

## Reasons for the current proposal
[rationale]: #reasons-for-the-current-proposal

Our reasons for preferring the current proposal are:

* Introducing a new ABI makes reliance on cross-language exception handling
  more explicit.
* `panic=abort` can be safely used with `extern "C unwind"` (there is no
  undefined behavior except with improperly used forced unwinding), but `extern
  "C"` has more optimization potential (eliding landing pads). Having two ABIs
  puts this choice in the hands of users.
  * Additionally, there are very few cases in which the current proposal
    permits the `panic=abort` runtime to introduce undefined behavior to a
    program that is well-defined under `panic=unwind`. The "option 3" proposal
    causes any foreign exception entering Rust to have undefined behavior under
    `panic=abort`.
* This design has simpler forward compatibility with alternate `panic!`
  implementations. Any well-defined cross-language unwinding will require shims
  to translate between the Rust unwinding mechanism and the natively provided
  mechanism. In this proposal, only `"C unwind"` boundaries would require shims.

[inside-rust-post]: https://blog.rust-lang.org/inside-rust/2020/02/27/ffi-unwind-design-meeting.html

# Prior art
[prior-art]: #prior-art

C++ as specified has no concept of "foreign" exceptions or of an underlying
exception mechanism. However, in practice, the C++ exception mechanism is the
"native" unwinding mechanism used by compilers.

On Microsoft platforms, when using MSVC, unwinding is always supported for both
C++ and C code; this is very similar to "option 3" described in [the
inside-rust post][inside-rust-post] mentioned [above][alternatives].

On other platforms, GCC, LLVM, and any related compilers provide a flag,
`-fexceptions`, for explicitly ensuring that stack frames have unwinding
support regardless of the language being compiled. Conversely,
`-fno-exceptions` removes unwinding support even from C++. This is somewhat
similar to how Rust's `panic=unwind` and `panic=abort` work for `panic!`
unwinds, and under the "option 3" proposal, the behavior would be similar for
foreign exceptions as well. In the current proposal, though, such foreign
exception support is not enabled by default with `panic=unwind` but requires
the new `"C unwind"` ABI.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!-- TODO -->

 `catch_unwind` (link to PR discussion)

coercions

# Future possibilities
[future-possibilities]: #future-possibilities

<!-- TODO -->

more interactions w/ foreign exceptions

shims will be required if Rust changes its unwind mechanism
