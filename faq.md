## FAQ questions

### Why use a new ABI? Why not an attribute?

- Unlike ABIs, there is no syntax for assigning attributes to function
  pointers. The ABI must be part of the type system so that callers of function
  pointers know whether or not the function may unwind.

### What should I do if I want to have C++ functions that unwind into Rust? [DRAFT; TODO]

- mark declarations in Rust `extern "C unwind"`
- Do *not* use `catch_unwind` in calling code
- compile w/ `panic = unwind`
- Check compiler, linker, & platform documentation (TODO: expand on this)
- Re-test for every compiler update
- (TODO: other caveats?)

### How does cross-language unwinding differ from cross-language `setjmp`/`longjmp`?

- `setjmp`/`longjmp` across Rust frames is currently intended to have
  well defined behavior as long as those frames do not contain
  destructors, although we don't have any documentation to that
  effect.
- When crossing frames that do contain destructors, the behavior of
  `longjmp` is [Undefined Behavior]; conversely, a primary goal of
  defining cross-language unwinding behavior is to support crossing
  frames with destructors.
- Rust does not have a concept of `Copy` for stack-frames, which would permit
  the compiler to check that `longjmp` may safely traverse those frames. Such a
  language feature [may be added in the future][centril-effects], but although
  it would be useful for `longjmp`, it would not be useful for unwinding.
- It should never be assumed that `drop` will be called for objects in
  intermediate frames traversed by a `longjmp`, but this may occur on certain
  platforms. Rust provides no guarantee either way (which is why this is
  considered [Undefined Behavior]). Cross-language unwind, however, will be
  defined such that `Drop` objects whose frames are unwound are guaranteed
  `drop`ed.
- Unwinding across Rust frames when `panic = abort` is currently undefined
  behavior, but we plan to define the behavior to cause the application to
  `abort`. The behavior of `setjmp`/`longjmp`, however, is independent of the
  `panic` runtime.
- unwinding involves the use of a personality function, which raises additional
  cross-language compatibility concerns; `setjmp`/`longjmp` does not.

[centril-effects]: https://github.com/Centril/rfc-effects/issues/11

### What new constraints does this ABI place on Rust's unwinding mechanism?

- Ideally: none. However, if Rust's default unwinding mechanism changes, a
  translation layer will be required to maintain the `C unwind` ABI.
- One more concern is how `panic = abort` will be handled; please refer to [the
  roadmap][roadmap-panic-abort] for details.

[roadmap-panic-abort]: roadmap/c-unwind-abi.md#panic--abort
[Undefined Behavior]: /spec-terminology.md#UB
