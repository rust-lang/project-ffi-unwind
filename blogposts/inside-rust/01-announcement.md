# Help the FFI-unwind project!

-- Kyle Strand and Niko Matsakis on behalf of the FFI-unwind project group --

The FFI-unwind project group, announced in [this RFC][rfc-announcement], is
working to extend the language to support unwinding that crosses FFI
boundaries.

We have been exploring the problem space, but we would like to begin making
some concrete decisions. Before publishing our first technical RFC, we are
extending a broader invitation to the Rust community to join our discussions
and help us guage the impact of the tradeoffs under consideration.

## What is unwinding?

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
