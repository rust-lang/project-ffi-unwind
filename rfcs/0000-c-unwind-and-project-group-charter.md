- Feature Name: `simple_c_panic_abi`
- Start Date: 2019-08-29
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

* First, to modify Rust functions with the "C" abi to abort on unwinding, as proposed
  in [rust-lang/rust#52652].
* Second, to introduce the "C unwind" ABI, with virtually all details
  (but not quite all) left as "to be determined".
    * The **intent** of this ABI, however, is to permit Rust panics to
      be "translated" into a "native" unwinding mechanism, though
      precisely which is dependent on the target platform or other
      compiler options.
        * (In practice, this translation is a no-op, since Rust panics
          use the native mechanism, but this is explicitly not
          required for the future.)
    * Folks currently using the "C" ABI to do unwinding can migrate to this new
      "C unwind" ABI, which captures their intentions. Their code is no more stable
      than it was, but they are on the right path.
* Finally, to create a "project group" with the purpose of designing subsequent
  RFCs to flesh out the details of the "C unwind" ABI
    * The "project group" term is newly introduced, but it corresponds
      to a kind of working group -- one with the goal of fleshing out
      a particular proposal or completing a project.
    * We aim to specify how "C unwind" works on major platforms
    * Goal is to enable Rust panics to propagate across native frames
    * And perhaps to enable native exceptions to propagate across Rust frames
    * But not to allow catching or throwing native exceptions from Rust code

# Motivation
[motivation]: #motivation

## We have no FFI ABI that permits unwinding, and we need one

Rust has long held that the `extern "C"` ABI does not permit
unwinding. Therefore, attempts to unwind through a `extern "C"`
barrier are considered [Undefined Behavior]. However, in practice, a
number of projects have found that "things work fine" so long as
certain conditions are met (for example, the frames that are to be
unwound do not contain destructors).

We have made a number of efforts to "tighten up" the behavior around
the C ABI and unwinding. For example, when users define a Rust
function that has the C ABI, we would like to make such functions
abort if unwinding actually occurs at runtime. Unfortunately, these
attempts have caused existing, working projects to break.

The problem here is not that the existing projects break *per se*: they
are relying on [Undefined Behavior] and that is to be expected as a
possibility. The problem is that there is no alternative available to
them that could allow them to keep working (even if they are
continuing to rely on behavior that is not yet fully specified).

## Specific use cases

We have seen a number of unwinding use cases in the wild. This section
details some of those use cases and their specific requirements.

### Lucet: unwind past empty native frames

The **[Lucet]** compiler generates native frames that invoke Rust
helpers. These frames are always invoked from some Rust helper.  From
time to time, they may invoke other Rust helpers, resulting in native
frames "sandwiched" between two Rust frames:

[Lucet]: https://www.fastly.com/blog/announcing-lucet-fastly-native-webassembly-compiler-runtime

```
[Rust root frame]
     |
     v
[Native frames]
     |
     v
[Rust helper function]
```

This Rust helper function may panic. Lucet would like to be able to
catch this panic from the Rust root frame. They would like to have the
Native frames "gracefully unwind". However, the native frames
themselves do not contain any destructors or landing pads -- they
simply need to be "skipped over", in effect.

Moreover, Lucet executes only on a narrow range of platforms:

* XXX details

### mozjpeg: XXX

The `mozjpeg-rust` crate provides an idiomatic Rust wrapper over the `libjpeg` library. When this library errors, it calls a user-provided callback that is not allowed to return. The documentation mentions that this callback either aborts or `longjmp` somewhere. The `mozjpeg-rust` crates implements this callback to `panic!` instead, unwinding from the Rust callback into C, and then from C back into Rust.

### lua bindings and longjmp

Although longjmp/setjmp are not directly in scope for this group,
there is an interesting interaction that was uncovered and is worth
documenting. On Windows targets specifically, longjmp is implemented
using the same "Structured Exception Handling" (SEH) mechanism that is
used for C++ exceptions and other sorts of errors. As a result, in
some versions of Rust at least, we found that attempts to longjmp over
Rust frames triggered unwinding mechanisms. Note that in general a
longjmp over a Rust frame that contains destructors is [Undefined
Behavior] -- however, we would like to support longjmp for Rust frames
that do not contain destructors. At minimum, this group should take
the behavior of longjmp into account; if it is easy to do, we may also
introduce mechanisms or RFCs that help to specify the use of longjmp
in particular cases.

## Our proposal

This RFC proposes a multi-prong plan to solve this problem:

* First, to modify Rust functions with the "C" abi to abort on unwinding, as proposed
  in [rust-lang/rust#52652].
* Second, to introduce the "C unwind" ABI, with virtually all details
  (but not quite all) left as "to be determined".
    * The **intent** of this ABI, however, is to permit Rust panics to
      be "translated" into a "native" unwinding mechanism, though
      precisely which is dependent on the target platform or other
      compiler options.
        * (In practice, this translation is a no-op, since Rust panics
          use the native mechanism, but this is explicitly not
          required for the future.)
    * Folks currently using the "C" ABI to do unwinding can migrate to this new
      "C unwind" ABI, which captures their intentions. Their code is no more stable
      than it was, but they are on the right path.
* Finally, to create a "project group" with the purpose of designing subsequent
  RFCs to flesh out the details of the "C unwind" ABI
    * The "project group" term is newly introduced, but it corresponds
      to a kind of working group -- one with the goal of fleshing out
      a particular proposal or completing a project.
    * We aim to specify how "C unwind" works on major platforms
    * Goal is to enable Rust panics to propagate across native frames
    * And perhaps to enable native exceptions to propagate across Rust frames
    * But not to allow catching or throwing native exceptions from Rust code

The first two goals work together.
The goal of introducing the "C unwind" ABI is simple: it allows
existing projects to migrate to the newer ABI and continue to work "as
well as they ever did" or better, since they will avoid certain optimizations
that are incorrect for their use cases. In the meantime, we can tighten up the rules
around the "C" ABI.

Simply taking that step, however, does not address the root problem of
"de facto" dependencies. We've already seen that projects can and do
rely on unwinding.

The majority of this RFC is spent on describing the "C unwind" ABI and
its details. We do note a few details about the "project group", but we
expect the "project group" to largely define its own structure.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The "C" ABI and unwinding

Functions declared with the "C" ABI are not permitted to unwind.  In
general, any attempt to unwind constitutes [Undefined Behavior], which
means that the program may do arbitrary things.

### Invoking foreign functions

This means, for example, that if you invoke a native "C" function and it
attempts to unwind (for example, because it invokes C++ code which contains a `throw`),
this result is [Undefined Behavior]:

```rust
extern "C" { fn fn_that_is_not_supposed_to_unwind(); }

fn foo() {
    // If `fn_that_is_not_supposed_to_unwind` should actually unwind, 
    // the result is undefined behavior.
    fn_that_is_not_supposed_to_unwind();
}
```

### Rust functions with C ABI

In the case of Rust functions declared with the "C" ABI, attempts to
unwind over the function boundary will result in an abort:

```rust
extern "C" fn foo() {
    // This panic will trigger an unwind that tries to pass over an extern "C" 
    // boundary. It will trigger an abort.
    panic!();
}
```

## The "C unwind" ABI and unwinding

In contrast, functions declared with the "C unwind" ABI **are**
permitted to unwind. The precise unwinding mechanism in use will
depend on the target platform, but the intent is to match the
mechanism used by other systems programming language implementations,
such as C++.

**WARNING:** The specifiation for the "C unwind" ABI is being actively
developed and is currently very incomplete. In particular, it is **not
currently possible to unwind between Rust and non-Rust frames in a
fully specified fashion**. If you would like to learn more about the
specification effort, or to get involved, please visit the repository
for the [ffi-unwind project], which contains a lot more details.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Platform dependent behavior of "C" and "C unwind"

In general, the behavior of both the "C" and the "C unwind" ABIs is
highly dependent on the target triple and other switches that are
given to the Rust compiler. Therefore, there isn't a lot that can be
said that is fully independent of the platform.

## Relationship between "C" and "C unwind"

There is no a priori relationship between the "C" and "C unwind" ABIs.
However, on most platforms, it is expected that the "C" ABI would be
"compatible" with the "C unwind" ABI. In other words, consider some C
function `foo` that does not unwind: on most platforms, such a
function could safely be invoked with either the "C" or the "C unwind"
ABI. 

However, we do not guarantee such interop: this leaves room for
platforms where unwinding is transmitted through a special return
value or other such mechanism. (XXX do any such platforms exist? There
is the C++ proposal)

## Typing and interconversion

Function pointer types declared with the "C unwind" and "C" ABIs are
generally incompatible, just as they would be with any other two ABIs
(e.g., "stdcall" and "C"). It is not possible, for example, to cast a
`extern "C" fn()` value into a `extern "C unwind" fn()` value. It is
however possible to convert between ABIs by using a closure expression
(which is then coerced into a standalone function):

```rust
fn convert(f: extern "C" fn()) -> extern "C unwind" fn() {
    return || f();
}
```

## Relationship between "C unwind" ABI and Rust's native panic unwinding

It is an explicit goal *not* to constrain or specify the unwinding
mechanism used by Rust functions with the standard Rust ABI. We do
require, however, that *some* mechanism exists to "convert" a Rust
panic into the native unwinding mechanism when crossing over the "C
unwind" ABI. Similarly, when a Rust function invokes a "C panic"
function, some mechanism must exist to "recover" a Rust panic on the
other side. In other words, a **Rust panic** must be able to unwind
across a "C unwind" boundary in a seamless fashion.

Therefore, the following program is guaranteed to work as expected:

```rust
fn catch() {
    let _ = catch_unwind(|| will_panic());
}

extern "C panic" fn will_panic() {
    panic!("Hello, world!");
}
```

**Implementation note.** In practice, Rust functions already use the native
unwinding mechanism, so this "interconversion" function is just a no-op. Moreover,
as of this RFC, we do not specify the "C unwind" ABI on any platforms at all. However,
if in the future:

* we specify the behavior of "C unwind" on some platform
* Rust's panic mechanism on that platform diverges from this behavior

then it would be required that some form of "interop conversion" be
defined as well.

# Workings of the "ffi-unwind" project group

## What is a "project group"?

In addition to introducing the "C unwind" ABI, this RFC also proposes
the creation of a new "project group" devoted to continued development
of this feature. The "project group" term has not previously been
used: it is intended to formalize a concept that has existed
informally for some time, under a number of names (including "working
group").

In general, a "project group" is basically a set of folks working on a
particular *project*. If you'll forgive some "retconning", examples of
other "project groups" might be the Unsafe Code Guidelines effort and
the work around defining const evaluation.

## Charter of the ffi-unwind project group

The ffi-unwind project group defined here has the following initial scope:

* to define the details of the "C unwind" ABI on major Tier 1 platforms
* in particular, to define with sufficient detail to enable the use cases
  described in the Motivation section of this RFC
  
Certain elements are definitively out of scope:

* The group does not intend to consider mechanisms to enable "interop"
  between Rust panics and exceptions from other languges. For example,
  we do not intend to permit Rust code to catch C++ exceptions, though
  we will have to consider what happens when a C++ exception unwinds
  past a `catch_unwind` boundary.
  
## Constraints and considerations

In its work, the project-group should consider various constraints and considerations:

* The possibility that C++ may adopt new unwinding mechanisms in the future.
* The possibility that Rust may alter its unwinding mechanism in the future -- in particular,
  be alert for implicit assumptions and dependencies.

## Relationship of the ffi-unwind project group to lang team

The group will define some (small) number of **shepherds**. These
folks are responsible for summarizing conversations and keeping the
lang team abreast of interesting developments. The initial shepherds of the
group are the following:

* [acfoltzer (Adam)](https://github.com/acfoltzer)
* [batmanaod (Kyle)](https://github.com/batmanaod)

In addition, the lang team is repsonsible for providing **liasons**, who are representatives
of the lang team that will try to stay abreast with the discussion. The initial
lang team liasons are the following:

* [nikmoatsakis (Niko)](https://github.com/nikmoatsakis)
* [joshtriplett (Josh)](https://github.com/joshtriplett)

## Project group roadmap and RFCs

The first step of the project group is to define a **roadmap**,
basically indicating which sets of things it will try to define and in
what order. You can see a draft version of the roadmap on the
repository. This roadmap will be presented to the lang team in
meetings. 

As an example roadmap item, it seems likely that the initial effort
will focus on defining the minimum needed to support the Lucet use
case, since that is a fairly simple example.

Once the project group feels it has completed work on some item in the
roadmap, that item will be converted into an RFC. 

To continue with the example roadmap item, we might open an RFC that
specifies certain details of the "C unwind" ABI for the platforms that
Lucet requires.

**Note:** The RFCs that result from the project group are already expected to have
undergone a fair amount of iteration. Therefore, it is likely that 

## Participation in the project group

Like any Rust group, the ffi-unwind project group intends to operate
in a public and open fashion and welcomes participation. Visit the
repository for more details.

# Drawbacks
[drawbacks]: #drawbacks

## Danger of de facto dependencies on implementation details

The primary drawback of this proposal is that we add an ABI, "C
unwind", whose interactions are not fully specified -- indeed, it is
so unspecified that the only specified interop is for one Rust
function to call another Rust function using that ABI. These
limitations might not be fully understood or communicated, resulting
in even greater level of "de facto" reliance on specific
implementation details.

We intend to mitigate this danger through the creation of the project
group, so that we ensure that we are actively stabilizing the details
that folks are relying on (or updating them, as needed).

## Unwinding is a complicated morass 

One danger is that fully specifying the details of how Rust
interoperates with unwinding on various platforms will be a long and
painful undertaking. We intend to mitigate this by focusing on
**specific use cases** and thus trying to avoid specifying more things
than those that are needed for those use cases. We also intend to
limit ourselves to major platforms to start.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The goal is to enable the current use-cases for unwinding panics through foreign code while providing safer behaviour towards bindings not expecting that, with minimal changes to the language.

We should try not to prevent the RFC 2699 or other later RFC from resolving the above disadvantages on top of this one.

- https://github.com/rust-lang/rfcs/pull/2699

The alternatives considered are:

1. Stabilize the [abort-on-FFI-boundary behavior](https://github.com/rust-lang/rust/issues/52652)
   without providing any mechanism to change this behavior. As a workaround for applications that would
   otherwise need this unwinding, we would recommend the use of wrappers to translate Rust panics to
   and from C ABI-compatible values at FFI boundaries, with a foreign control mechanism like
   `setjmp`/`longjmp` or C++ exceptions to skip or unwind segments of foreign stack.

   While using these types of wrappers will likely continue to be a recommendation for
   maximally-compatible code, it comes with a number of downsides:

   - Additional non-Rust code must be maintained.
   - `setjmp`/`longjmp` incur runtime overhead even when no unwinding is required, whereas many
     unwinding mechanisms incur runtime overhead only when unwinding.
   - Wrappers that must be present at Rust compile time are not suitable for applications with
     dynamic, generated code.

   If an application has enough control over its compilers and runtime environment to be assured
   that its foreign components are compatible with the unspecified Rust unwinding mechanism, these
   downsides can be avoided by allowing unwinding across FFI boundaries.

2. Address unwinding more thoroughly, perhaps through the introduction of additional `extern "ABI"`
   strings. This is the approach being pursued in #2699, and is widely seen as a better long-term
   solution than adding a single attribute. However, work in #2699 has stalled due to a number of
   thorny questions that will likely take significant time and effort to resolve, such as:

   - What should the syntax be for unwind-capable ABIs?
   - What are the type system implications for new ABI strings?
   - How should the semantics of an unwind-capable ABI be defined across different platforms?

   In the meantime, we are caught between wanting to fix the soundness bug in Rust, and not wanting
   to disrupt current development on a number of projects that depend on unwinding. Adding an unwind
   attribute means that we can address those current needs right away, and then transition to
   #2699's eventual solution by converting the attribute into a deprecated proc-macro.

3. Using an attribute on function definitions and declarations to indicate that unwinding should be
   allowed, regardless of the ABI string. This would be easy to implement, as there is currently
   such an attribute in unstable Rust. An attribute is not a complete solution, though, as there is
   no current way to syntactically attach an attribute to a function pointer type (see
   https://github.com/rust-lang/rfcs/pull/2602). We considered making all function pointers
   unwindable without changing the existing syntax, because Rust currently does not emit `nounwind`
   for calls to function pointers, however this would require changes to the language reference that
   would codify inconsistency between function pointers and definitions/declarations.

# Prior art
[prior-art]: #prior-art

XXX Not written. Things that could be summarized:

* How does go handle unwinding across its FFI?
* Windows SEH and COM presumably try to cross languages -- at least cite it, perhaps?
* would be great for someone to add this

# Unresolved questions
[unresolved-questions]: #unresolved-questions

These questions must be resolved before stabilizing the "C unwind"
ABI.

## Are there platforms where `"C unwind"` should be forbidden altogether?

This would probably be platforms that have no dominant "native" notion
of unwinding that we plan to support. I imagine this may be the case
on many embedded platforms, for example. Presuambly such platforms
would also generally require `-Cpanic=abort`. Do any such platforms
exist?

## When should we stabilize the "C unwind" ABI?

The main motivation for "C unwind" is to give existing libraries a
"migration path" so that we can "tighten" the behavior for the "C"
ABI. This likely implies that "C unwind" must be stabilized at the
same time that the "abort shims" from [rust-lang/rust#52652] are
stabilized, since otherwise those libraries would no longer be able to
compile on stable, which is precisely the sort of regression we are
attempting to avoid. However, it may be that the library authors are
satisfied with building on nightly, in which case we could leave "C
unwind" as a nightly-only feature longer, which helps to avoid the
risk of undue "de facto" dependency.

It is the position of this RFC that the primary determinant here are
the needs of those libraries that are relying on the "C" ABI to
support some measure of unwinding.  We have a number of existing
features that fit the "general precedent" of stable syntax with
[unspecified] semantics -- ranging from the layout of many Rust data
types to the behavior of raw pointers -- and so this form of
stabilization is not unprecedented.

# Future possibilities
[future-possibilities]: #future-possibilities

The project group will of course explore the full specification of the "C unwind" ABI.

Other future possibilities 

[Undefined Behavior]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html
[ffi-unwind project]: https://github.com/nikomatsakis/project-ffi-unwind
[rust-lang/rust#52652]: https://github.com/rust-lang/rust/issues/52652
