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
<!-- XXX mention `mozjpeg` or others? -->
But this code is not merely theoretically incorrect; it has also been broken in
practice by changes to Rust's code generation.
<!-- XXX be more specific, or leave this vague? -->

There are, however, some valid use cases that are impossible or impracticable
without some form of cross-language unwinding.
<!-- XXX examples? Lucet? XXX what would an MVP be? -->

## Defaults matter: Finalizing the behavior of `extern "C"`

### The discussion so far

Defining a specific unwinding implementation is overly limiting for Rust's development, so the project group is not authorized to simply declare ... <!-- XXX -->

### Gathering metrics with a `rustc` fork

## A soundness hole in stable Rust

[rfc-announcement]: https://github.com/rust-lang/rfcs/pull/2797
