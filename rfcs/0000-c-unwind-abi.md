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
Rust frames; on Windows, `longjmp` is implemented via [forced
unwinding](forced-unwinding). The current `rustc` implementation makes it safe
to `longjmp` across Rust frames without `Drop` types, but this is not formally
specified in an RFC or by the Reference.

The desire for this feature has been previously discussed on other RFCs,
including [#2699](https://github.com/rust-lang/rfcs/pull/2699) and [#2753](https://github.com/rust-lang/rfcs/pull/2573).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When declaring an external function that may unwind, such as an entrypoint to a
C++ library, use `"C unwind"`:

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
[forced-unwinding]: #forced-unwinding

This is a special kind of unwinding used to implement `longjmp` on Windows and
`pthread_exit` in `glibc`. A brief explanation is provided in [this Inside Rust
blog post][inside-rust-forced]. This RFC distinguishes forced unwinding from
other types of foreign unwinding.

Since language features and library functions implemented using forced
unwinding on some platforms use other mechanisms on other platforms, Rust code
cannot rely on forced unwinding to invoke destructors (calling `drop` on `Drop`
types). In other words, a forced unwind operation on one platform will simply
deallocate Rust frames without true unwinding on other platforms.

This RFC specifies that it is undefined behavior to cross Rust frames with
destructors via such language features and library functions, regardless of the
platform.

[inside-rust-forced]: https://blog.rust-lang.org/inside-rust/2020/02/27/ffi-unwind-design-meeting.html#forced-unwinding

## Changes to `extern "C"` behavior

Prior to this RFC, any unwinding operation that crossed an `extern "C"`
boundary, either from a `panic!` "escaping" from a Rust function defined with
`extern "C"` or by entering Rust from another language via an entrypoint
declared with `extern "C"`, caused undefined behavior.

This RFC retains most of that undefined behavior, with two exceptions:

* With the `panic=unwind` runtime, `panic!` will cause an `abort` if it would
  otherwise "escape" from a function defined with `extern "C"`.
* Forced unwinding is safe with `extern "C"` as long as no frames with
  destructors (i.e. `Drop` types) are unwound. This is to keep behavior of
  `pthread_exit` and `longjmp` consistent across platforms.

## Interaction with `panic=abort`

If a non-forced foreign unwind would enter a Rust frame via an `extern "C
unwind"` ABI boundary, but the Rust code is compiled with `panic=abort`, the
unwind will be caught and the process aborted.

There are some types of unwinding that are not guaranteed to cause the program
to abort with `panic=abort`, though:

* Forced unwinding: Rust provides no mechanism to catch this type of unwinding.
  This is safe with either the `"C"` ABI or the new `"C unwind"` ABI, as long
  as none of the unwound Rust frames contain destructors.
* Unwinding from another language into Rust if the entrypoint to that language
  is declared with `extern "C"` (contrary to the guidelines above): this is
  always undefined behavior.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This table shows the behavior of an unwinding operation reaching each type of
ABI boundary (function declaration or definition). "UB" stands for undefined
behavior. `"C"`-like ABIs are `"C"` itself but also related ABIs such as
`"system"`.

| panic runtime  | ABI          | `panic`-unwind                        | Unforced foreign unwind |
| -------------- | ------------ | ------------------------------------- | ----------------------- |
| `panic=unwind` | `"C unwind"` | unwind                                | unwind                  |
| `panic=unwind` | `"C"`-like   | abort                                 | UB                      |
| `panic=abort`  | `"C unwind"` | `panic!` aborts                       | abort                   |
| `panic=abort`  | `"C"`-like   | `panic!` aborts (no unwinding occurs) | UB                      |

Regardless of the panic runtime, ABI, or platform, the interaction of Rust
frames with C functions that deallocate frames (i.e. functions that may use
forced unwinding on specific platforms) is specified as follows:

* **When deallocating Rust frames without destructors:** frames are safely
  deallocated; no undefined behavior
* **When deallocating Rust frames with destructors:** undefined behavior

No subtype relationship is defined between functions or function pointers using
different ABIs. This RFC also does not define coercions between `"C"` and
`"C unwind"`.

As noted in the [summary][summary], if a Rust frame containing `catch_unwind` is unwound by a
foreign exception, the behavior is undefined for now.

# Drawbacks
[drawbacks]: #drawbacks

Forced unwinding is treated as universally unsafe across frames with
destructors, but on some platforms it could theoretically be well-defined. As
noted [above](forced-unwind), however, this would make the UB inconsistent
across platforms, which is not desirable.

This design imposes some burden on existing codebases (mentioned
[above][motivation]) to change their `extern` annotations to use the new ABI.

Having separate ABIs for `"C"` and `"C unwind"` may make interface design more
difficult, especially since this RFC [postpones][future-possibilities]
introducing coercions between function types using different ABIs.

A single ABI that "just works" with C++ (or any other language that may throw
exceptions) would be simpler to learn and use than two separate ABIs.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

As explained in [this Inside Rust blog post][inside-rust-requirements], we have
several requirements for any cross-language unwinding design:

* We need to ensure that Rust may in the future adopt an alternate unwinding
  mechanism; our design must continue working despite such a change.
* With the `panic=abort` runtime, we must be able to optimize the binary size by removing most code related
  to unwinding.
* Changing the panic runtime from `panic=unwind` to `panic=abort` should generally not
  cause undefined behavior. (Note that the current proposal does not fully
  satisfy this criterion.)
* We cannot change the ABI of functions in the `libc` crate, because this would
  be a breaking change: function pointers of different ABIs have different
  types.

[inside-rust-requirements]: https://blog.rust-lang.org/inside-rust/2020/02/27/ffi-unwind-design-meeting.html#requirements-for-any-cross-language-unwinding-specification

## Other proposals discussed with the lang team
[alternatives]: #other-proposals-discussed-with-the-lang-team

Two other potential designs have been discussed in depth; they are
explained in [this Inside Rust blog post][inside-rust-proposals]. The design in this
RFC is referred to as "option 2" in that post.

"Option 1" in that blog post only differs from the current proposal in the
behavior of a forced unwind across a `"C unwind"` boundary under `panic=abort`.
Under the current proposal, this type of unwind is permitted, allowing
`longjmp` and `pthread_exit` to behave "normally" with both the `"C"` and the
`"C unwind"` ABI across all platforms regardless of panic runtime. If there are
destructors in the unwound frames, this results in undefined behavior. Under
"option 1", however, all foreign unwinding, forced or unforced, is caught at
`"C unwind"` boundaries under `panic=abort`, and the process is aborted. This
gives `longjmp` and `pthread_exit` surprising behavior on some platforms, but
avoids that cause of undefined behavior in the current proposal.

The other proposal in the blog post, "option 3", is dramatically different. In
that proposal, foreign exceptions are permitted to cross `extern "C"`
boundaries, and no new ABI is introduced.

[inside-rust-proposals]: https://blog.rust-lang.org/inside-rust/2020/02/27/ffi-unwind-design-meeting.html#three-specific-proposals

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
  * This optimization could be made available with a single ABI by means of an
    annotation indicating that a function cannot unwind (similar to C++'s
    `noexcept`). However, Rust does not yet support annotations for function
    pointers, so until that feature is added, such an annotation could not be
    applied to function pointers.
* This design has simpler forward compatibility with alternate `panic!`
  implementations. Any well-defined cross-language unwinding will require shims
  to translate between the Rust unwinding mechanism and the natively provided
  mechanism. In this proposal, only `"C unwind"` boundaries would require shims.

# Prior art
[prior-art]: #prior-art

C++ as specified has no concept of "foreign" exceptions or of an underlying
exception mechanism. However, in practice, the C++ exception mechanism is the
"native" unwinding mechanism used by compilers.

On Microsoft platforms, when using MSVC, unwinding is always supported for both
C++ and C code; this is very similar to "option 3" described in [the
inside-rust post][inside-rust-proposals] mentioned [above][alternatives].

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

The behavior of `catch_unwind` when a foreign exception encounters it is
currently [left undefined][reference-level-explanation]. We would like to
provide a well-defined behavior for this case, which will probably be either to
let the exception pass through uncaught or to catch some or all foreign
exceptions.

# Future possibilities
[future-possibilities]: #future-possibilities

As mentioned [above][rationale], shims will be required if Rust changes its
unwind mechanism.

We may want to provide more means of interaction with foreign exceptions. For
instance, it may be possible to provide a way for Rust to catch C++ exceptions
and rethrow them from another thread. See the [discussion
above][unresolved-questions] about the behavior of `catch_unwind`.

Coercions between `"C unwind"` function types (such as function pointers) and
the other ABIs are not part of this RFC. However, they will probably be
indispensible for API design, so we plan to provide them in a future RFC.
