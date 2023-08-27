# Request for Stabilization

This is a request for *full* stabilization of the `c_unwind` feature, [RFC-2945][rfc-text]. The behavior of non-`"Rust"`, non-`unwind` ABIs (such as `extern "C"`) will be modified to close soundness holes caused by permitting stack-unwinding to cross FFI boundaries that do not support unwinding.

## Summary

When using `panic=unwind`, if a Rust function marked `extern "C"` panics (and that panic is not caught), the runtime will now abort.

Previously, the runtime would simply attempt to unwind the caller's stack, but the behavior when doing so was undefined, because `extern "C"` functions are optimized with the assumption that they cannot unwind (i.e. in `rustc`, they are given the LLVM `nounwind` annotation).

This affects existing programs. If a program relies on a Rust panic "escaping" from `extern "C"`:
* It is currently unsound.
* Once this feature is stabilized, the program will crash when this occurs, whereas previously it may have appeared to work as expected.
* Replacing `extern "x"` with `extern "x-unwind"` will produce the intended behavior without the unsoundness.

The behavior of function calls using `extern "C"` is unchanged; thus, it is still undefined behavior to call a C++ function that throws an exception using the `extern "C"` ABI, even when compiling with `panic=unwind`.

## Changes since the RFC

None.

### Unresolved questions

This feature does not affect the behavior of `pthread_exit` or `longjmp`, so stabilization will neither break existing code using these features nor provide any soundness guarantee.

## Tests

### `"C"` guaranteed to abort on panic

[This test][abort-on-panic] verifies that an `extern "C"` function that `panic!`s will always abort if the panic would otherwise "escape".

## Documentation

The documentation PRs for this feature did not distinguish between the initial partially-stabilized behavior and the fully-stabilized behavior, so they already include the modifications to non-`unwind` ABIs.

* [The Rustonomicon PR][nomicon] has already been merged.
* [The Reference PR][reference] is still awaiting approval.

I've also [opened an issue][book-issue] to discuss adding a brief note about this feature in the Book.

<!-- links -->
[rfc-text]: https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md
[abort-on-panic]: https://github.com/rust-lang/rust/blob/master/tests/ui/panics/abort-on-panic.rs
[nomicon]: https://github.com/rust-lang/nomicon/pull/365
[reference]: https://github.com/rust-lang/reference/pull/1226
[book-issue]: https://github.com/rust-lang/book/issues/3729
