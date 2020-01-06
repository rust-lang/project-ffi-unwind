# Announcing the first FFI-unwind project design meeting

-- Kyle Strand and Niko Matsakis on behalf of the FFI-unwind project group --

The FFI-unwind project group, announced in [this RFC][rfc-announcement], is
working to extend the language to support unwinding that crosses FFI
boundaries.

We have reached our first technical decision point, on a question we have been
discussing internally for quite a while.  This blog post lays out the arguments
on each side of the issue and invites the Rust community to join us at the
upcoming meeting to help finalize our decision, which will be formalized and
published as our first language-change RFC. This RFC will propose an "MVP"
specification for well-defined cross-language unwinding.

<!-- XXX below, a "lang team meeting" was mentioned (I have now merged the
relevant paragraph with the one above); are these the same? I.e., will the
design meeting be a lang team meeting? -->

## The question: should the `"C"` ABI permit unwinding?

The core question that we would like to decide is whether the `"C"` ABI, as
defined by Rust, should permit unwinding.

This is not a question we expected to be debating. We've long declared that
unwinding through Rust's `"C" ABI is undefined behavior. In part, this is
because nobody had spent the time to figure out what the correct behavior would
be, or how to implement it, although (as we'll see shortly) there are other
good reasons for this choice. 

In any case, in PR #65646, @Amanieu proposed that we could, in fact, simply
define the behavior of unwinding across `"C"` boundaries. In discussing this,
discovered that the question of whether the `"C" ABI should permit unwinding was
less clear-cut than we had assumed.

## Background: What is unwinding?

Exceptions are a familiar control flow mechanism in many programming languages.
They are particularly commonplace in managed languages such as Java, but they
are also implemented in the C++ language, which introduced destructors (known
as "finalizers" in some languages) to the world of unmanaged systems
programming.

When an exception is thrown, the runtime _unwinds_ the stack, essentially
traversing it backwards and calling clean-up or error-recovery code such as
destructors or `catch` blocks.

It is well known that Rust does not have exceptions as such. But Rust _does_
support unwinding! There are two scenarios that will cause unwinding to occur:

* Using FFI, Rust can call functions in other languages (such as C++) that can
  unwind the stack.
  * There are some special cases where C libraries can actual cause unwinding.
    For instance, on Microsoft platforms, `longjmp` is implemented with
    unwinding.
* By default, Rust's own `panic!()` unwinds the stack.

## Towards a stable unwinding MVP

The behavior of `panic!()` is generally well-defined: in most cases, it will
terminate the thread or the entire program unless it is stopped by
`catch_unwind()`.

When C++ code invokes Rust code, however, the behavior of a `panic!()` unwind
that "escapes" from Rust into C++ is explicitly undefined. Similarly, when Rust
calls a foreign function that unwinds, the behavior once the unwind operation
encounters Rust frames is undefined.

Currently, Rust and C++ running in the same process space share the same
runtime and the same unwinding mechanism. The ability to unwind across language
boundaries has been used successfully in some crates despite the danger of
exploiting undefined behavior.
<!-- TODO mention `mozjpeg` or others? -->
But this code is not merely theoretically incorrect; it has also been broken in
practice by changes to Rust's code generation.
<!-- TODO be more specific, or leave this vague? -->

There are, however, some valid use cases that are impossible or impracticable
without some form of cross-language unwinding.
<!-- TODO examples? Lucet? -->

The minimum feature set needed to safely enable these uses cases is a
well-defined way to let Rust `panic`s "escape" `extern` Rust functions, and
permit them to re-enter Rust code via non-Rust functions declared in Rust with
`extern`. The project's first technical RFC will propose a specification for
these abilities.

## Defaults matter: Finalizing the behavior of `extern "C"`

As usual when adding new features to Rust, we would like to avoid breaking
changes if possible. We have discussed numerous possible non-breaking
_additions_ to the language that would enable users to explicitly select the
desired behavior for unwinding across FFI boundaries. But since the existing
`extern "C"` function declaration syntax (and any other non-Rust ABI
specification) is not explicit, we believe that selecting a default behavior
for cross-FFI `panic`s would be preferable to leaving it undefined.

The original plan, pre-existing the project group by several years, is to make
Rust functions defined with `extern "C"` immediately `abort` if a `panic` would
otherwise unwind out of the function. This was actually implemented and the
behavior introduced to stable Rust twice, once in <!-- TODO version --> and
again in <!-- TODO version -->. The change was reverted both times, however,
because it breaks the only currently-working method for unwinding from Rust
into other languages.

<!-- TODO other options -->

### The discussion so far

Defining a specific unwinding implementation is overly limiting for Rust's
development, so the project group is not authorized to simply declare ...
<!-- TODO -->

### Gathering metrics with a `rustc` fork

## A soundness hole in stable Rust

[rfc-announcement]: https://github.com/rust-lang/rfcs/pull/2797
