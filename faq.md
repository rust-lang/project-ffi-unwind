## FAQ questions

- Why use a new ABI? Why not an attribute?
  - Unlike ABIs, there is no syntax for assigning attributes to function
    pointers. The ABI must be part of the type system so that callers of
    function pointers know whether or not the function may unwind.
- What should I do if I want to have C++ functions that unwind into Rust? [DRAFT; TODO]
  - mark declarations in Rust `extern "C unwind"`
  - Do *not* use `catch_unwind` in calling code
  - compile w/ `panic = unwind`
  - Check compiler, linker, & platform documentation (TODO: expand on this)
  - Re-test for every compiler update
  - (TODO: other caveats?)
- How does cross-language unwinding differ from cross-language
  `setjmp`/`longjmp`?
  - `setjmp`/`longjmp` across Rust frames is currently guaranteed not to have
    undefined behavior or to `abort`
  - It should never be assumed that `drop` will be called for objects in
    intermediate frames traversed by a `longjmp`, but this may occur on certain
    platforms. Rust provides no guarantee either way. Conversely, on platforms
    where unwinding is supported, it will always call `drop`.
  - Rust does not have a concept of `Copy` for stack-frames, which would permit
    the compiler to check that `longjmp` may safely traverse those frames. Such
    a language feature may be added in the future, but although it would be
    useful for `longjmp`, it would not be useful for unwinding.
  - `setjmp`/`longjmp` does not involve data transfer (the exception object)
  - unwinding involves the use of a personality function, which raises
    additional cross-language compatibility concerns; `setjmp`/`longjmp` does
    not
- What new constraints does this ABI place on Rust's unwinding mechanism?
  - Ideally: none. However, if Rust's default unwinding mechanism changes, a
    translation layer will be required to maintain the `C unwind` ABI.
