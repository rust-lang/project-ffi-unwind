# Announcing the first FFI-unwind project design meeting

-- Kyle Strand and Niko Matsakis on behalf of the FFI-unwind project group --

The FFI-unwind project group, announced in [this RFC][rfc-announcement], is
working to extend the language to support unwinding that crosses FFI
boundaries.

We have reached our first technical decision point, on a question we have been
discussing internally for quite a while. This blog post lays out the arguments
on each side of the issue and invites the Rust community to join us at the
upcoming meeting to help finalize our decision, which will be formalized and
published as our first language-change RFC. This RFC will propose an "MVP"
specification for well-defined cross-language unwinding.

<!-- TODO below, a "lang team meeting" was mentioned (I have now merged the
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

There are many different possible designs, but the most important question is
to decide between two alternatives:

* Permit any functions using the `"C"` ABI to unwind
* Add a new ABI (`"C unwind"`) that permits unwinding; the `"C"` ABI is
  specified as the system ABI but where unwinding is UB
  <!-- TODO Isn't the "default" design to _abort_ when Rust code would
  otherwise expose an unwind? -->

## Background: what is unwinding?

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

Currently, when foreign (non-Rust) code invokes Rust code, the behavior of a
`panic!()` unwind that "escapes" from Rust is explicitly undefined. Similarly,
when Rust calls a foreign function that unwinds, the behavior once the unwind
operation encounters Rust frames is undefined.

## Requirements for any cross-language unwinding specification

* Unwinding between Rust functions (and in particular unwinding of Rust panics)
  may not necessarily use the system unwinding mechanism
  * In practice, we do use the system mechanism today, but we would like to
    reserve the freedom to change this
* If you enable `-Cpanic=abort`, we are able to optimize the size of binaries
  to remove most code related to unwinding.
  * In extreme cases, it should be possible to remove **all** traces of
    unwinding, when you know that it will not occur.
  * In practice, most "C" functions are never expected to unwind (because they
    are written in C, for example, and not in C++).
    * However, because unwinding is now part of most system ABIs, even C
      functions can unwind -- most notably, `pthread_cancel` can cause all
      manner of C functions to unwind, including common functions like `read`
      and `write`.  (For a complete list, search for "cancellation points" in
      the [pthreads man
      page](http://man7.org/linux/man-pages/man7/pthreads.7.html).)
* Changing the behavior from `-Cpanic=unwind` to `-Cpanic=abort` should not
  cause Undefined Behavior.
  * However, this may not be tenable, or at least not without making binaries
    much larger. See the discussion below for more details.
  * It may, of course, cause programs to abort that used to execute
    successfully. This could occur if a panic would've been caught and
    recovered.

## Gathering metrics with a `rustc` fork

## A soundness hole in stable Rust

[rfc-announcement]: https://github.com/rust-lang/rfcs/pull/2797
