# "C unwind" ABI

## Summary

Functions that use the plain `"C"` ABI are **not permitted to unwind**.
Doing so is [Undefined Behavior], which means that the compiler is free
to assume it cannot happen. The results are therefore unpredictable.
It is always a bug.

We are adding therefore a new ABI `"C unwind"`. This ABI can be used
to indicate functions that follow the C ABI and which **may** unwind.
The precise unwinding mechanism that such functions use will depend on
the target definition. In general, though, it is meant to match the
"native" unwinding mechanism of the target: typically whatever C++
compilers use.

**Warning:** We are still in the process of fully specifying when and
how "unwinding interop" works between native functions and Rust. This
roadmap item alone **only** adds the ABI string to Rust. All details
of how that ABI strings are implemented for various targets are still
considered [To Be Defined]. As a result, programs that rely on those
details are not truly stable; you may find that the details change as
Rust evolves. Eventually, though, we do intend to define many (but not
all) aspects of how Rust panics and native unwinding interoperate.
Moreover, we guarantee that unwinding will **not** result in
[Undefined Behavior] and in particular not [LLVM-UB].

## The goal

In practice, there are a number of crates that rely on unwinding
across a "C" ABI today -- even though that is [Undefined
Behavior]. Creating the "C unwind" ABI means that those crates can
migrate to this ABI and better express their intent.  It does not (in
and of itself) make the behavior of those crates defined -- they are
still relying on [To Be Defined] details. But it *does* mean that they
will no longer trigger [Undefined Behavior].

## Panic = abort

In order to safely call functions that may unwind, a Rust function must have
a landing pad. Typically, `panic = abort` prevents landing pads from being
generated, which would make the behavior of `"C unwind"`
[LLVM-undefined][LLVM-UB]. To make this behavior well defined, we would like to
require landing-pad generation for any function calling a `"C unwind"`
function, even when compiling with `panic = abort`. These landing pads would of
course `abort` the application rather than propagate the unwind.

[Undefined Behavior]: /spec-terminology.md#UB
[LLVM-UB]: /spec-terminology.md#LLVM-UB
[To Be Defined]: /spec-terminology.md#TBD
