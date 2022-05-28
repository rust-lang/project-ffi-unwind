# Request for Stabilization

<!-- Note: waiting on https://github.com/rust-lang/rust/pull/97235 -->
<!-- Note: need to finish this documentation: https://github.com/BatmanAoD/reference/tree/c-unwind-documentation -->

This is a request for stabilization of the `c_unwind` feature,
[RFC-2945][rfc-text].

## Summary

This feature enables ABI strings ending in `-unwind`; specifically:

- `"C-unwind"`
- `"cdecl-unwind"`
- `"stdcall-unwind"`
- `"fastcall-unwind"`
- `"vectorcall-unwind"`
- `"thiscall-unwind"`
- `"aapcs-unwind"`
- `"win64-unwind"`
- `"sysv64-unwind"`
- `"system-unwind"`

The behavior for unforced unwinding (the typical case) is specified in [this
table from the RFC][rfc-table], and has not been changed since the RFC was
accepted. To summarize:

Each ABI is mostly equivalent to the same ABI without `-unwind`, except that
the behavior is defined to be safe when an unwinding operation (`panic` or C++
style exception) crosses the ABI boundary. For `panic=unwind`, this is a valid
way to let exceptions from one language unwind the stack in another language
without terminating the process (as long as the exception is caught in the same
language from which it originated); for `panic=abort`, this will typically
abort the process immediately.

Conversely, the corresponding ABIs without `-unwind` will no longer permit
unwinding across their ABI boundary. This has previously been undefined
behavior but was permitted in practice (albeit with soundness holes). Their
interaction with unwinding will still remain mostly undefined, except that a
Rust `panic` when `panic=unwind` will now cause the process to abort if it
would otherwise escape from a non-Rust FFI boundary without `-unwind`.

## examples

### Rust `panic` with `"C-unwind"`

```rust
#[no_mangle]
extern "C-unwind" fn example() {
    panic!("Uh oh");
}
```

This function now permitted to unwind C++ stack frames.

```
[Rust function with `catch_unwind`, which stops the unwinding]
      |
     ...
      |
[C++ frames]
      |                           ^
      | (calls)                   | (unwinding
      v                           |  goes this
[Rust function `example`]         |  way)
      |                           |
      +--- rust function panics --+
```

If the C++ frames have objects, their destructors will be called.

### C++ `throw` with `"C-unwind"`

```rust
#[link(...)]
extern "C-unwind" {
    // A C++ function that may throw an exception
    fn may_throw();
}

#[no_mangle]
extern "C-unwind" fn rust_passthrough() {
    let b = Box::new(5);
    unsafe { may_throw(); }
    println!("{:?}", &b);
}
```

A C++ function with a `try` block may invoke `rust_passthrough` and `catch` an
exception thrown by `may_throw`.

```
[C++ function with `try` block that invokes `rust_passthrough`]
      |
     ...
      |
[Rust function `rust_passthrough`]
      |                            ^
      | (calls)                    | (unwinding
      v                            |  goes this
[C++ function `may_throw`]         |  way)
      |                            |
      +--- C++ function throws ----+
```

If `may_throw` does throw an exception, `b` will be dropped. Otherwise, `5`
will be printed.

### `panic` can be stopped at an ABI boundary

```rust
#[no_mangle]
extern "C" fn assert_nonzero(input: u32) {
    assert!(input != 0)
}
```

Previously, if `assert_nonzero` were called with the argument `0` with a
`panic=unwind` runtime, the behavior would be undefined, because `assert!`
would `panic`, and the behavior of a `panic` crossing a non-Rust ABI boundary
was undefined. With the `c_unwind` feature, the runtime is guaranteed to
(safely) abort the process if this occurs.

## Tests

<!-- requirements, from the stabilization guide:
+ A summary, showing examples (e.g. code snippets) what is enabled by this feature.
- Links to test cases in our test suite regarding this feature and describe the feature's behavior on encountering edge cases.
- Links to the documentations (the PRs we have made in the previous steps).
- Any other relevant information.
- The resolutions of any unresolved questions if the stabilization is for an RFC.
-->

<!-- links -->
[rfc-text]: https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md
[rfc-table]: https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md#abi-boundaries-and-unforced-unwinding
