- Feature Name: `c_unwind`
- Start Date: 2019-10-15
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC adds a new Application Binary Interface (ABI) string `"C unwind"` that
is only available on certain supported targets that provide a native C ABI that
supports unwinding (e.g. `C` + `-fexceptions`).

Functions with this ABI unwind using the native unwinding mechanism of the platform.

This RFC adds support of this ABI for some tier-1 targets, but it is expected
that support for this ABI will be added to more targets in the future, either in
other RFCs that build on this one, or in FCPs.

# Motivation
[motivation]: #motivation

On most mainstream platforms, the native platform ABI supports unwinding. This
allows native applications on those platforms to expose APIs that unwind, and to
call libraries whose APIs unwind. The `"C unwind"` ABI string allows defining
Rust functions that can be called by other native libraries on that platform
(written in Rust, or in other programming languages that support the ABI):

```rust
// In Rust crate A:
extern "C unwind" fn foo() { 
    panic!("hello world");
}
```

as well as declaring and calling functions with such an ABI from Rust:

```rust
// In Rust crate B, linked with Rust crate A:
extern "C unwind" {
    fn foo();
}

fn bar() {
    let result = std::panic::catch_unwind(|| unsafe { foo() });
    assert!(result.is_err());
}
```

One example of a platform with a native ABI that supports unwinding is Linux,
where section "6.2 Unwind Library Interface" of the [x86-64 ps-ABI] and the
[Itanium ABI] specification document the ABI.

One example of a library API that uses unwinding on "its" API is [pthreads],
which stands for POSIX threads. This library is available on many platforms. If
you go to the section of its documentation called ["Cancellation
points"][pthreads], there are dozens of functions like `open()`, `fsync()`,
`sleep()` and so on marked as such. All these functions are currently imported
into Rust by the `libc` library using the `extern "C"` ABI which is not allowed
to unwind. Sadly, all of their declarations are unsound if a user uses
`pthreads_cancel` (also exposed by `libc`) to cancel a thread, because that
might cause any of these functions to unwind. As a consequence, [this example
program](https://play.rust-lang.org/?version=nightly&mode=release&edition=2018&gist=e2d5a366754e6fd2d7406e2dcee8f7c0)
has undefined behavior:

```rust
struct DropGuard;
impl Drop for DropGuard {
    fn drop(&mut self) {
        println!("unwinding foo");
    }
}

fn foo() {
    let _x = DropGuard;
    println!("thread started");
    std::thread::sleep(std::time::Duration::new(1, 0));
    println!("thread finished");
}

fn main() {
    let handle0 = std::thread::spawn(foo);
    std::thread::sleep(std::time::Duration::new(0, 10_000));
    unsafe { 
        let x = pthread_cancel(handle0.as_pthread_t());
        assert_eq!(x, 0);
    }
    handle0.join();
}
```

Depending on how you see it, the culprit here is either the `pthread_cancel`
call, which will cause the `sleep` function to panic, or the declaration of the
`sleep` function within the `libc` crate, which uses the `extern "C"` ABI which
is not allowed to unwind.

So the motivation of this RFC is that right now it is not possible to write Rust
code that interfaces with native libraries that can unwind using the platforms
native unwinding ABI, and such libraries are ubiquotous on all major platforms
(recall, the `p` in `pthreads` stands for POSIX).

[x86-64 ps-ABI]: https://github.com/hjl-tools/x86-psABI/wiki/X86-psABI
[itanium]: https://itanium-cxx-abi.github.io/cxx-abi/abi.html
[pthreads]: http://man7.org/linux/man-pages/man7/pthreads.7.html

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC introduces a new ABI string called `"C unwind"` on some specific
targets. If this ABI is not available for a target, Rust will error at
compile-time saying so. This RFC adds support of this ABI for some tier-1
targets, but it is expected that support for this ABI will be added to more
targets in the future, either in other RFCs that build on this one, or in FCPs.

Like for all other ABI strings, one can use this ABI string:

  * to define a function in Rust: `extern "C unwind" fn foo() { }`
  * to declare a function in Rust: `extern "C unwind" { fn foo(); }`
  * in function pointer types: `let _: extern "C unwind" fn = foo;`

Similarly to the `"Rust"` ABI string, `"C unwind"` functions can unwind:

* if the `"C unwind"` function is defined in Rust, it unwinds the stack, and the
  panic can be caught with `catch_unwind`.
  
* if the `"C unwind"` function is not defined in Rust, it unwinds the stack, but
  whether such unwindings can always, sometimes, or never be caught with
  `catch_unwind` or not is target-dependent. If some of these unwindings can be
  caught, their value is then of type `std::panic::ForeignPanic` (that is, the
  `Result::Err(Any)` that `catch_unwind` returns can be downcasted to such a
  type).
      * if the "panic-strategy" is set to `abort`, calling a `"C unwind"`
  function that unwinds either aborts the program or 
  

When a `"C unwind"` function that is defined in Rust unwinds, the Rust panic is
translated to a native panic at the function boundary. Calling a function with
the `"C unwind"` ABI from Rust translates the native unwind into a Rust panic.

When the native panics originate in Rust code, the translation back to Rust
panics is lossless. 

However, when the native panic 



# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Currently, Rust assumes that an external function imported with `extern "C" {
... }` (or another ABI other than `"Rust"`) cannot unwind, and Rust will abort if a panic would propagate out of a
Rust function with a non-"Rust" ABI ("`extern "ABI" fn`") specification. Under this RFC,
functions with the `"C panic"` ABI string
instead allow Rust panic unwinding to proceed through that function
boundary using Rust's normal unwind mechanism. This may potentially allow Rust
code to call non-Rust code that calls back into Rust code, and then allow a
panic to propagate from Rust to Rust across the non-Rust code.

The Rust unwind mechanism is intentionally not specified here. Catching a Rust
panic in Rust code compiled with a different Rust toolchain or options is not
specified. Catching a Rust panic in another language or vice versa is not
specified. The Rust unwind may or may not run non-Rust destructors as it
unwinds. Propagating a Rust panic through non-Rust code is unspecified;
implementations that define the behavior may require target-specific options
for the non-Rust code, or this feature may not be supported at all.

For the purposes of the type system, `"C panic"` is considered a totally distinct ABI string from
`"C"`. While there may be some circumstances for which an `extern "C" fn` in place
of an `extern "C panic" fn` (or vice-versa) would be useful, this introduces questions of subtyping and variance
that are beyond the scope of this RFC. This restrictive approach is forwards-compatible with more
permissive typing in future work like #2699.

# Drawbacks
[drawbacks]: #drawbacks

- Only works as long as the foreign code supports the same unwinding mechanism as Rust. (Currently, Rust and C++ code compiled for ABI-compatible backends use the same mechanism.)

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The goal is to enable the current use-cases for unwinding panics through foreign code while providing safer behaviour towards bindings not expecting that, with minimal changes to the language.

We should try not to prevent the RFC 2699 or other later RFC from resolving the above disadvantages on top of this one.

- https://github.com/rust-lang/rfcs/pull/2699

The alternatives considered are:

1. Stabilize the [abort-on-FFI-boundary behavior](https://github.com/rust-lang/rust/issues/52652)
   without providing any mechanism to change this behavior. As a workaround for applications that would
   otherwise need this unwinding, we would recommend the use of wrappers to translate Rust panics to
   and from C ABI-compatible values at FFI boundaries, with a foreign control mechanism like
   `setjmp`/`longjmp` or C++ exceptions to skip or unwind segments of foreign stack.

   While using these types of wrappers will likely continue to be a recommendation for
   maximally-compatible code, it comes with a number of downsides:

   - Additional non-Rust code must be maintained.
   - `setjmp`/`longjmp` incur runtime overhead even when no unwinding is required, whereas many
     unwinding mechanisms incur runtime overhead only when unwinding.
   - Wrappers that must be present at Rust compile time are not suitable for applications with
     dynamic, generated code.

   If an application has enough control over its compilers and runtime environment to be assured
   that its foreign components are compatible with the unspecified Rust unwinding mechanism, these
   downsides can be avoided by allowing unwinding across FFI boundaries.

2. Address unwinding more thoroughly, perhaps through the introduction of additional `extern "ABI"`
   strings. This is the approach being pursued in #2699, and is widely seen as a better long-term
   solution than adding a single attribute. However, work in #2699 has stalled due to a number of
   thorny questions that will likely take significant time and effort to resolve, such as:

   - What should the syntax be for unwind-capable ABIs?
   - What are the type system implications for new ABI strings?
   - How should the semantics of an unwind-capable ABI be defined across different platforms?

   In the meantime, we are caught between wanting to fix the soundness bug in Rust, and not wanting
   to disrupt current development on a number of projects that depend on unwinding. Adding an unwind
   attribute means that we can address those current needs right away, and then transition to
   #2699's eventual solution by converting the attribute into a deprecated proc-macro.

3. Using an attribute on function definitions and declarations to indicate that unwinding should be
   allowed, regardless of the ABI string. This would be easy to implement, as there is currently
   such an attribute in unstable Rust. An attribute is not a complete solution, though, as there is
   no current way to syntactically attach an attribute to a function pointer type (see
   https://github.com/rust-lang/rfcs/pull/2602). We considered making all function pointers
   unwindable without changing the existing syntax, because Rust currently does not emit `nounwind`
   for calls to function pointers, however this would require changes to the language reference that
   would codify inconsistency between function pointers and definitions/declarations.

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

# Future possibilities
[future-possibilities]: #future-possibilities

TODO

- https://github.com/rust-lang/rfcs/pull/2699

- `unwind(abort)`
- non-"C" ABIs
