- Feature Name: `"C unwind" ABI`
- Start Date: 2019-04-03
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)
- Project group: [FFI-unwind][project-group]

[project-group]: https://github.com/rust-lang/project-ffi-unwind

# Summary
[summary]: #summary

We introduce a new ABI string, `"C unwind"`, to enable unwinding from other
languages (such as C++) into Rust frames and from Rust into other languages.

Additionally, we define the behavior for a limited number of
previously-undefined cases when an unwind operation reaches a Rust function
boundary with a non-`"Rust"`, non-`"C unwind"` ABI.

As part of this specification, we introduce the term ["Plain Old Frame"
(POF)][POF-definition]. These are frames that may be safely deallocated with
`longjmp`.

This RFC does not define the behavior of `catch_unwind` in a Rust frame being
unwound by a foreign exception. This is something the [project
group][project-group] would like to specify in a future RFC; as such, it is
"TBD" (see ["Unresolved questions"][unresolved-questions]).

# Motivation
[motivation]: #motivation

There are some Rust projects that need cross-language unwinding to provide
their desired functionality. One major example is Wasm interpreters, including
the Lucet and Wasmer projects.

There are also existing Rust crates (notably, wrappers around the `libpng` and
`libjpeg` C libraries) that `panic` across C frames. The safety of such
unwinding relies on compatibility between Rust's unwinding mechanism and the
native exception mechanisms in GCC, LLVM, and MSVC.

Additionally, there are libraries such as `rlua` that rely on `longjmp` across
Rust frames; on Windows, `longjmp` is implemented via [forced
unwinding][forced-unwinding]. The current `rustc` implementation makes it safe
to `longjmp` across Rust [POFs][POF-definition] (frames without `Drop` types),
but this is not formally specified in an RFC or by the Reference.

The desire for this feature has been previously discussed on other RFCs,
including [#2699][rfc-2699] and [#2753][rfc-2753].

## Key design goals

As explained in [this Inside Rust blog post][inside-rust-requirements], we have
several requirements for any cross-language unwinding design.

The ["Analysis of key design goals"][analysis-of-design-goals] section analyzes
how well the current design satisfies these constraints.

* **Changing from panic=unwind to panic=abort cannot cause UB:** We
  wish to ensure that choosing `panic=abort` doesn't ever create
  undefined behavior (relate to `panic=unwind`), even if one is
  relying on a library that triggers a panic or a foreign exception.
* **Optimization with panic=abort:** when using `-Cpanic=abort`, we
  wish to enable as many code-size optimizations as possible. This
  means that we shouldn't have to generate unwinding tables or other
  such constructs, at least in most cases.
* **Preserve the ability to change how Rust panics are propagated when
  using the Rust ABI:** Currently, Rust panics are propagated using
  the native unwinding mechanism, but we would like to keep the
  freedom to change that.
* **Enable Rust panics to traverse through foreign frames:** Several
  projects would like the ability to have Rust panics propagate
  through foreign frames.  Those frames may or may not register
  destructors of their own with the native unwinding mechanism.
* **Enable foreign exceptions to propagate through Rust frames:**
  Similarly, we would like to make it possible for C++ code (or other
  languages) to raise exceptions that will propagate through Rust
  frames "as if" they were Rust panics (i.e., running destrutors or,
  in the case of `unwind=abort`, aborting the program).
* **Enable error handling with `longjmp`:** As mentioned above, some existing
  Rust libraries use `longjmp`. Despite the fact that `longjmp` on Windows is
  [technically a form of unwinding][forced-unwinding], using `longjmp` across
  Rust [POFs][POF-definition] [is safe][longjmp-pr] with the current
  implementation of `rustc`, and we want to specify that this will remain safe.
* **Do not change the ABI of functions in the `libc` crate:** Some `libc`
  functions may invoke `pthread_exit`, which uses [a form of
  unwinding][forced-unwinding] in the GNU libc implementation. Such functions
  must be safe to use with the existing `"C"` ABI, because changing the types
  of these functions would be a breaking change. 

  [inside-rust-requirements]: https://blog.rust-lang.org/inside-rust/2020/02/27/ffi-unwind-design-meeting.html#requirements-for-any-cross-language-unwinding-specification
  [longjmp-pr]: https://github.com/rust-lang/rust/pull/48572

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Functions declared with `extern "C"` are generally not permitted to unwind
(with some very narrow exceptions, described [below][forced-unwinding]).
This means, for example, that such a function cannot throw an uncaught C++
exception, and it also cannot invoke Rust code that may panic (unless that
panic is caught with `catch_unwind`).

When declaring an external function that may unwind, such as an entrypoint to a
C++ library, use `"C unwind"` instead:

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
  may_throw();
}
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

## "Plain Old Frames"
[POF-definition]: #plain-old-frames

A "POF", or "Plain Old Frame", is defined as a frame that can be trivially
deallocated; that is, returning from or unwinding a POF cannot cause any
observable effects. This means that POFs do not contain any pending destructors
(live `Drop` objects) or `catch_unwind` calls.

The terminology is intentionally akin to [C++'s "Plain Old Data"
types][cpp-POD-definition], which are types that, among other requirements, are
trivially destructible (their destructors do not cause any observable effects,
and may be elided as an optimization).

Rust frames that do contain pending destructors or `catch_unwind` calls are
called non-POFs.

Note that a non-POF may _become_ a POF, for instance if all `Drop` objects are
moved out of scope, or if its only `catch_unwind` call is in a code path that
will not be executed.

[cpp-POD-definition]: https://en.cppreference.com/w/cpp/named_req/PODType

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

This RFC specifies that, regardless of the platform or the ABI string (`"C"` or
`"C unwind"`), any platform features that may rely on forced unwinding are:

* _undefined behavior_ if they cross non-[POFs][POF-definition]
* _defined behavior_ when all unwound frames are POFs

As an example, this means that Rust code can (indirectly, via C) invoke
`longjmp` using the "C" ABI, and that `longjmp` can unwind or otherwise cross
Rust [POFs][POF-definition]. If those Rust frames are not POFs, then invoking
`longjmp` would be undefined behavior (and hence a bug).

[inside-rust-forced]: https://blog.rust-lang.org/inside-rust/2020/02/27/ffi-unwind-design-meeting.html#forced-unwinding

## Changes to `extern "C"` behavior

Prior to this RFC, any unwinding operation that crossed an `extern "C"`
boundary, either from a `panic!` "escaping" from a Rust function defined with
`extern "C"` or by entering Rust from another language via an entrypoint
declared with `extern "C"`, caused undefined behavior.

This RFC retains most of that undefined behavior, with two exceptions:

* With the `panic=unwind` runtime, `panic!` will cause an `abort` if it would
  otherwise "escape" from a function defined with `extern "C"`.
* Forced unwinding is safe with `extern "C"` as long as only
* [POFs][POF-definition] are unwound. This is to keep behavior of
  `pthread_exit` and `longjmp` consistent across platforms.

## Interaction with `panic=abort`

If a non-forced foreign unwind would enter a Rust frame via an `extern "C
unwind"` ABI boundary, but the Rust code is compiled with `panic=abort`, the
unwind will be caught and the process aborted.

There are some types of unwinding that are not guaranteed to cause the program
to abort with `panic=abort`, though:

* Forced unwinding: Rust provides no mechanism to catch this type of unwinding.
  This is safe with either the `"C"` ABI or the new `"C unwind"` ABI, as long
  as only [POFs][POF-definition] are unwound.
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

* **When deallocating Rust [POFs][POF-definition]:** frames are safely
  deallocated; no undefined behavior
* **When deallocating Rust non-POFs:** undefined behavior

No subtype relationship is defined between functions or function pointers using
different ABIs. This RFC also does not define coercions between `"C"` and
`"C unwind"`.

As noted in the [summary][summary], if a Rust frame containing a pending
`catch_unwind` call is unwound by a foreign exception, the behavior is
undefined for now.

# Drawbacks
[drawbacks]: #drawbacks

Forced unwinding is treated as universally unsafe across
[non-POFs][POF-definition], but on some platforms it could theoretically be
well-defined. As noted [above](forced-unwind), however, this would make the UB
inconsistent across platforms, which is not desirable.

This design imposes some burden on existing codebases (mentioned
[above][motivation]) to change their `extern` annotations to use the new ABI.

Having separate ABIs for `"C"` and `"C unwind"` may make interface design more
difficult, especially since this RFC [postpones][future-possibilities]
introducing coercions between function types using different ABIs.

A single ABI that "just works" with C++ (or any other language that may throw
exceptions) would be simpler to learn and use than two separate ABIs.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Other proposals discussed with the lang team
[alternatives]: #other-proposals-discussed-with-the-lang-team

Two other potential designs have been discussed in depth; they are
explained in [this Inside Rust blog post][inside-rust-proposals]. The design in this
RFC is referred to as "option 2" in that post.

"Option 1" in that blog post only differs from the current proposal in the
behavior of a forced unwind across a `"C unwind"` boundary under `panic=abort`.
Under the current proposal, this type of unwind is permitted, allowing
`longjmp` and `pthread_exit` to behave "normally" with both the `"C"` and the
`"C unwind"` ABI across all platforms regardless of panic runtime. If
[non-POFs][POF-definition] are unwound, this results in undefined behavior.
Under "option 1", however, all foreign unwinding, forced or unforced, is caught
at `"C unwind"` boundaries under `panic=abort`, and the process is aborted.
This gives `longjmp` and `pthread_exit` surprising behavior on some platforms,
but avoids that cause of undefined behavior in the current proposal.

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
  * The single-ABI proposal ("option 3") causes any foreign exception entering
    Rust to have undefined behavior under `panic=abort`, whereas the current
    proposal does not permit the `panic=abort` runtime to introduce undefined
    behavior to a program that is well-defined under `panic=unwind`.
  * This optimization could be made available with a single ABI by means of an
    annotation indicating that a function cannot unwind (similar to C++'s
    `noexcept`). However, Rust does not yet support annotations for function
    pointers, so until that feature is added, such an annotation could not be
    applied to function pointers.
* This design has simpler forward compatibility with alternate `panic!`
  implementations. Any well-defined cross-language unwinding will require shims
  to translate between the Rust unwinding mechanism and the natively provided
  mechanism. In this proposal, only `"C unwind"` boundaries would require shims.

## Analysis of key design goals
[analysis-of-design-goals]: #analysis-of-design-goals

This section revisits the key design goals to assess how well they
are met by the proposed design.

### Changing from panic=unwind to panic=abort cannot cause UB

This constraint is met:

* Unwinding across a "C" boundary is UB regardless
    of whether one is using `panic=unwind` or `panic=abort`.
* Unwinding across a "C unwind" boundary is always defined,
    though it is defined to abort if `panic=abort` is used.
* Forced exceptions behave the same regardless of panic mode.

### Optimization with panic=abort

Using this proposal, the compiler is **almost always** able to reduce
overhead related to unwinding when using panic=abort. The one
exception is that invoking a "C unwind" ABI still requires some kind
of minimal landing pad to trigger an abort. The expectation is that
very few functions will use the "C unwind" boundary unless they truly
intend to unwind -- and, in that case, those functions are likely
using panic=unwind anyway, so this is not expected to make much
difference in practice.

### Preserve the ability to change how Rust panics are propagated when using the Rust ABI

This constraint is met. If we were to change Rust panics to a
different mechanism from the mechanism used by the native ABI,
however, there would have to be a conversion step that interconverts
between Rust panics and foreign exceptions at "C unwind" ABI
boundaries.

### Enable Rust panics to traverse through foreign frames

This constraint is met.

### Enable foreign exceptions to propagate through Rust frame

This constraint is partially met: the behavior of foreign exceptions
with respect to `catch_unwind` is currently undefined, and left for
future work.

### Enable error handling with `longjmp`

This constraint is met: `longjmp` is treated the same across all platforms, and
is safe as long as only [POFs][POF-definition] are deallocated.

### Do not change the ABI of functions in the `libc` crate

This constraint is met: `libc` functions will continue to use the `"C"` ABI.
`pthread_exit` will be treated the same across all platforms, and will be safe
as long as only [POFs][POF-definition] are deallocated. 

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

## Attributes on nightly Rust

Currently, nightly Rust provides attributes, `#[unwind(allowed)]` and
`#[unwind(abort)]`, for making the behavior of `panic` crossing a `"C"` ABI
boundary well defined.
<!-- TODO explain why new ABI string is preferable to attributes -->

## Prior RFCs and other discussions

There were two previous RFCs, [#2699][rfc-2699] and [#2753][rfc-2753], that
attempted to introduce a well-defined way for uwnding to cross FFI boundaries.

<!-- TODO other discussions:
Tickets:
* https://github.com/rust-lang/rust/issues/58794
* https://github.com/rust-lang/rust/issues/52652
* https://github.com/rust-lang/rust/issues/58760
* https://github.com/rust-lang/rust/pull/55982

Discourse:
https://internals.rust-lang.org/t/unwinding-through-ffi-after-rust-1-33/9521?u=batmanaod
-->

[rfc-2699]: https://github.com/rust-lang/rfcs/pull/2699
[rfc-2753]: https://github.com/rust-lang/rfcs/pull/2573

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The behavior of `catch_unwind` when a foreign exception encounters it is
currently [left undefined][reference-level-explanation]. We would like to
provide a well-defined behavior for this case, which will probably be either to
let the exception pass through uncaught or to catch some or all foreign
exceptions.

Within the context of this RFC and in discussions among members of the
[FFI-unwind project group][project-group], this class of formally-undefined
behavior which we plan to define at later date is referred to as "TBD
behavior".

# Future possibilities
[future-possibilities]: #future-possibilities

The [FFI-unwind project group][project-group] intends to remain active at least
until all ["TBD behavior"][unresolved-questions] is defined.

We may want to provide more means of interaction with foreign exceptions. For
instance, it may be possible to provide a way for Rust to catch C++ exceptions
and rethrow them from another thread. Such a mechanism may either be
incorporated into the functionality of `catch_unwind` or provided as a separate
language or standard library feature.

Coercions between `"C unwind"` function types (such as function pointers) and
the other ABIs are not part of this RFC. However, they will probably be
indispensible for API design, so we plan to provide them in a future RFC.

As mentioned [above][rationale], shims will be required if Rust changes its
unwind mechanism.
