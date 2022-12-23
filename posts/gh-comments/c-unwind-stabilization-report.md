# Request for Stabilization

This is a request for *partial* stabilization of the `c_unwind` feature,
[RFC-2945][rfc-text]. The newly-introduced ABIs will be stabilized, but the
changes in behavior to existing ABIs (such as `extern "C"`) will remain behind
the `c_unwind` feature gate.

## Summary

This stabilization enables ABI strings ending in `-unwind`; specifically:

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

For initial stabilization, *no change* will be made to the existing ABIs. The
existing ABIs have soundness holes, though, which will be fixed when the
behavior described in the RFC is stabilized. Thus, we would like to stabilize
the remainder of the `c_unwind` feature soon. We are splitting the
stabilization into separate releases to give users time to adopt the new ABIs.

## Examples

### Rust `panic` with `"C-unwind"`

```rust
#[no_mangle]
extern "C-unwind" fn example() {
    panic!("Uh oh");
}
```

This function is now permitted to unwind C++ stack frames.

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

## Changes since the RFC

The implementation is as specified in the RFC, but there have been a few
considerations not directly addressed in the text of the RFC.

### Additional ABI strings

The RFC, as written, only introduces the `"C-unwind"` ABI string; the others
[listed above](#summary) were introduced later. Their semantics are exactly as
one would expect: they are identical to their non-`unwind` counterparts except
when an unwind reaches their ABI boundary, in which case the behavior is the
same as the RFC specifies for `"C-unwind"`.

Similarly, the new rules for `extern "C"` regarding unwinding are now applied
to all non-`unwind` ABI strings other than `"Rust"`.

### Double unwinding

[PR #92911][pr-double-unwind] ensured that the following scenarios both safely
abort the program:

* multiple foreign (e.g. C++) exceptions
* a `panic` occurring while a foreign exception is unwinding the stack, or
  vice-versa

### Mixing panic modes

We addressed a [soundness hole][issue-mixed-panic] regarding the behavior when
a foreign exception enters the Rust runtime via a crate compiled with
`panic=unwind`, but escapes into a crate compiled with `panic=abort`. We
decided to [prohibit][pr-fix-mixed-panic] any crate from being linked with the
`panic=abort` runtime if it has both of the following characteristics:

* It contains a call to an `-unwind` foreign function or function pointer
* It was compiled with `panic=unwind`

Note: `cargo` will automatically unify all crates to use the same `panic`
runtime, so this prohibition does not apply to projects compiled with `cargo`.

[PR #97235][pr-fix-mixed-panic] implemented this prohibition.

### Unresolved questions

None of the [unresolved questions][rfc-unresolved] have been resolved.
Specifically, the following are still considered undefined behavior:

* Letting a foreign exception unwind a Rust frame that calls `catch_unwind`
* Calling `pthread_exit` or `longjmp` in such a way that Rust frames are
  deallocated

## Tests

### Codegen

There is [a folder of codegen tests][codegen-unwind] for the new ABI strings.

Additional codegen tests are:

* [panic=abort][codegen-extra-1]
* [extern-exports][codegen-extra-2]
* [extern-imports][codegen-extra-3]

### `"C"` guaranteed to abort on panic

[This test][abort-on-panic] verifies that an `extern "C"` function that
`panic!`s will always abort if the panic would otherwise "escape".

### Full examples

Full example projects mixing Rust code with non-Rust code are:

* [a `panic` escaping into C][panic-into-c]
* [a C++ exception escapig into Rust][cpp-throw-into-rust]
* [double unwinding][double-unwind]

### Open PR

[PR #97235][pr-fix-mixed-panic] introduces tests for the [mixed-panic-modes
issue](#mixing-panic-modes).

## Documentation

Here are documentation PRs for

* [the Reference][reference]
* [the Rustonomicon][nomicon]

I have not yet suggested any changes for the Book, but it may be appropriate to
add something like this:

> Usually you want the non-unwind versions of ABI strings, but if you are
> messing around with FFI and unwinding, refer to the Nomicon.

I don't think Rust by Example needs to be updated.

<!-- links -->
[rfc-text]: https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md
[rfc-table]: https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md#abi-boundaries-and-unforced-unwinding
[rfc-unresolved]: https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md#unresolved-questions
[pr-fix-mixed-panic]: https://github.com/rust-lang/rust/pull/97235/files
[pr-double-unwind]: https://github.com/rust-lang/rust/pull/92911
[issue-mixed-panic]: https://github.com/rust-lang/rust/issues/96926
[codegen-unwind]: https://github.com/rust-lang/rust/tree/master/src/test/codegen/unwind-abis
[codegen-extra-1]: https://github.com/rust-lang/rust/blob/master/src/test/codegen/unwind-and-panic-abort.rs
[codegen-extra-2]: https://github.com/rust-lang/rust/blob/master/src/test/codegen/unwind-extern-exports.rs
[codegen-extra-3]: https://github.com/rust-lang/rust/blob/master/src/test/codegen/unwind-extern-imports.rs
[panic-into-c]: https://github.com/rust-lang/rust/tree/master/src/test/run-make-fulldeps/c-unwind-abi-catch-lib-panic
[cpp-throw-into-rust]: https://github.com/rust-lang/rust/tree/master/src/test/run-make-fulldeps/foreign-exceptions
[double-unwind]: https://github.com/rust-lang/rust/tree/master/src/test/run-make-fulldeps/foreign-double-unwind
[abort-on-panic]: https://github.com/rust-lang/rust/blob/master/src/test/ui/panics/abort-on-panic.rs
[nomicon]: https://github.com/rust-lang/nomicon/pull/365
[reference]: https://github.com/rust-lang/reference/pull/1226
