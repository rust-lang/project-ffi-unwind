# "C unwind" ABI

## Summary

Functions that use the plain `"C"` ABI are **not permitted to unwind**.
Doing so is "undefined behavior", which means that the compiler is free
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
roadmap item alone **only** adds the ABI -- it does not define what
should happen if unwinding actually occurs, and hence **even if you
use this ABI**, unwinding across a `"C unwind"` ABI barrier is **still
undefined behavior**. However, it is our **intent** to define that
behavior in the future (and this is what the other roadmap items
are all about). 

In practical terms, the effect of using `"C unwind"` right now is that
we will tell LLVM that the function "may unwind". We will also not add
intentional shims that abort the program if unwinding occurs. (With a
`"C"` ABI, we sometimes do both of those things.)

Further, in practical terms, you can use the `"C unwind"` ABI today to
enable a Rust panic to propagate across native frames if you like --
but your program is relying on unspecified and undefined behavior
which **likely will change** across Rust stable releases. Therefore,
you will have to keep up. Effectively, you're on a nightly release,
even though you're using only stable syntax. (The same is true for
many aspects of unsafe code.)

## Panic = abort

In order to safely call functions that may unwind, a Rust function must have
a landing pad. Typically, `panic = abort` prevents landing pads from being
generated, which would make the behavior of `"C unwind"`
[LLVM-undefined][LLVM-UB]. To make this behavior well defined, we would like to
require landing-pad generation for any function calling a `"C unwind"`
function, even when compiling with `panic = abort`. These landing pads would of
course `abort` the application rather than propagate the unwind.

[LLVM-UB]: spec-terminology.md#LLVM-undefined-behavior-or-LLVM-UB
