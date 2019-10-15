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

## Platforms APIs that unwind: Rust calls "C unwind"

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

## Unwinding from a Rust callback in C: "C unwind" calls Rust

Some C libraries like `mozjpeg` take callbacks using the platform native ABI and
expect these callbacks to unwind through C back into the user application to
report errors.

JIT compilers like [Lucet][lucet] and [Weld][weld]: TODO (I'm not sure I understood what these programs do; need some help here)
  * they generate code that calls other code using the C ABI
  * all the code except the ABI is implemented in Rust, so they want to be able to unwind through the ABI
  * the `"Rust"` call ABI is too "unstable" for their uses, and they prefer to use the more stable platform ABI

[lucet]: https://github.com/fastly/lucet	
[weld]: https://www.weld.rs/

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
    * Since `"C unwind"` is a super-set of `"C"`, there are implicit coercions
      from `extern "C"` function types to `extern "C unwind"` function types.
      These are analogous to the coercion rules for safe `fn` types to `unsafe
      fn` types.

Similarly to the `"Rust"` ABI string, `"C unwind"` functions can unwind:

* if the `"C unwind"` function is defined in Rust, it unwinds the stack, and the
  panic can be caught with `catch_unwind`.
  
* if the `"C unwind"` function is not defined in Rust, it unwinds the stack, but
  whether such unwindings can always, sometimes, or never be caught with
  `catch_unwind` or not is target-dependent. If some of these unwindings that do
  not originate in Rust can be caught, their value is then of type
  `std::panic::ForeignPanic` (that is, the `Result::Err(Any)` that
  `catch_unwind` returns can be downcasted to such a type).

The Rust panic ABI is unspecified. When a `"C unwind"` function that is defined
in Rust unwinds, the Rust panic is translated to a "native Rust panic" which
conforms to the native ABI. When a panic originates in Rust unwinds through a
`"C unwind"` function call back into Rust, the translation is guaranteed to be
lossless if the panic was not modified. That is, this behaves as if the panic
would have never left Rust.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The Rust panic ABI is unspecified. 

This RFC introduced the `"C unwind"` ABI string for the following tier-1 targets:

* `x86_64-unknown-linux-gnu`
* `x86_64-pc-windows-msvc`
* `x86_64-apple-darwin` 

On all other targets, usage of the `"C unwind"` generates a compilation error. 

This ABI does not specify the Rust unwinding ABI for these targets. Instead,
panics crossing function boundaries with the `"C unwind"` ABI are translated
back and forth from the Rust unwinding ABI to the "native Rust panic ABI" which
comforms to the platform ABI. This translation might incur a cost on some
platforms, and this cost might change over time as the Rust panic ABI evolves.

`"C unwind"` functions can unwind:

* if the `"C unwind"` function is defined in Rust, it unwinds the stack, and the
  panic can be caught with `catch_unwind`.
  
* if the `"C unwind"` function is not defined in Rust, it unwinds the stack. 
  * whether such unwindings can always, sometimes, or never be caught with
  `catch_unwind` or not is target-dependent. If some of these unwindings that do
  not originate in Rust can be caught, their value is then of type
  `std::panic::ForeignPanic` (that is, the `Result::Err(Any)` that
  `catch_unwind` returns can be downcasted to such a type).
    * if during unwinding with a native exception Rust panics, the program `abort`s.
    * if `panic=abort` the behavior is target-dependent.

The type `std::panic::ForeignPanic` is only available on selected targets and it is opaque. 
It does however allow re-raising the foreign exception using `resume_unwind` as follows:

```rust
let x: std::panic::ForeignPanic;
std::panic::resume_unwind(x);
```

The following implicit coercion rules are added for coercing `extern "C"`
function pointer types into `extern "C unwind"` types (note: the variance of
`fn(T) -> U` is contravariant for `T` and covariant for `U`):

```rust
extern "C" fn c_fn0(x: extern "C" fn()) { ... }
let a0: extern "C" fn(extern "C" fn()) = c_fn0; // OK(covariance)
let b0: extern "C unwind" fn(extern "C" fn()) = a0; // OK(covariance)
let c0: extern "C" fn(extern "C unwind" fn()) = a0; // ERROR(contravariance)

extern "C" fn c_fn1(x: extern "C unwind" fn()) { ... }
let a1: extern "C" fn(extern "C unwind" fn()) = c_fn1; // OK(covariance)
let b1:extern "C unwind" fn(extern "C unwind" fn()) = a1; // OK(covariance)
let c1:extern "C unwind"  fn(extern "C" fn()) = a1; // OK(contravariance)

extern "C" fn c_fn2() -> extern "C" fn() { ... }
let a2: extern "C" fn() -> extern "C" fn() = c_fn2; // OK(covariance)
let b2: fn() -> extern "C" fn() = a2; // OK(covariance)
let c2: extern "C" fn() -> fn() = a2; // OK(covariance)

extern "C" fn c_fn3() -> extern "C unwind" fn() { ... }
let a3: extern "C" fn() -> extern "C unwind" fn() = c_fn3; // OK(covariance)
let b3: extern "C" fn() -> extern "C" fn() = a3; // ERROR(covariance)
let c3: extern "C unwind" fn() -> extern "C unwind" fn() = a3; // OK(covariance)
```

## "C unwind" on `x86_64-unknown-linux-gnu` and `x86_64-apple-darwin`

These platforms native Rust panics conform to the Itanium ABI. The high 4 bytes
of the `exception_class` are `Rust` (the string is not null-terminated).

In Rust, 

* native "foreign" exceptions are all caught by `catch_unwind`, and according to
  the Itanium ABI the behavior of Rust programs that modify these exception
  objects is undefined. If a "foreign" exception reaches a thread boundary, the
  program `abort`s. If `-C panic=abort`, a "foreign" exception that unwinds into
  Rust `abort`s the program.

* "forced" unwindings are not caught by `catch_unwind`; if a "forced" unwinding
  reaches a thread boundary, the program `abort`s. Note: "forced" exceptions are
  used by `longjmp` and thread cancellation in `pthread`s. If `-C panic=abort`
  "forced" exceptions that unwind into Rust do not `abort` the program, but
  unwind the stack instead, and if doing so would require a destructor to run,
  the behavior is undefined (e.g. a `longjmp` that skips a destructor is UB).
  
> *Note*: The Itanium C++ ABI documents that, for C++, a Rust foreign exception can be
> caught by either `catch(...)` or by `catch(__rust_exception)` if
> `__rust_exception` is available in the `<exception>` header. If during unwinding
> due to a Rust foreign exception, a C++ program throws, the behavior is
> undefined.

## "C unwind" on `x86_64-pc-windows-msvc` 

TBD.

# Drawbacks
[drawbacks]: #drawbacks

TBD.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Multiple alternatives have been considered:

1. Do not add a feature for this. This would require users to write shims for
   all foreign APIs that could unwind to stop unwinding before entering Rust. It
   was decided in [abort-on-FFI-boundary
   behavior](https://github.com/rust-lang/rust/issues/52652) that the usability
   cost of this solution was too high and a better solution was neede.

2. Add a "C panic" ABI string that unwinds with the Rust ABI. This solution
   constraints the Rust unwinding implementation to be compatible with the C
   ABI, and does not solve the problem of interfacing with libraries using the
   platform's unwinding ABI.
   
3. Specify the Rust unwinding ABI on some platforms to match the native ABI and
   add a "C panic" ABI. The main drawback is that this limits the changes that
   could be made to the Rust unwinding ABI and the trade-offs of such a
   limitation are not clear. This solution isn't zero cost either, since Rust
   code that does not need to use "C unwind" would be paying a constraint on its
   panic ABI for a feature that it does not use.

4. Using an `#[unwind(allowed)]` attribute on `extern "C"` functions. This
   solution treats function pointers to functions that can or cannot unwind via
   a single function pointer type. That is, there is no way to avoid the
   overhead of unwinding in, e.g., code-size, for those function pointers for
   which the optimizer cannot statically prove that they always call functions
   that do not unwind.

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

# Future possibilities
[future-possibilities]: #future-possibilities

If the Rust unwinding ABI were to change in the future in such a way that the
translation cost from it to the "native Rust exception ABI", we could add a
toolchain option to allow users to select between different implementations of
the Rust `panic=unwind` strategy. For example, via `-C panic=unwind -C
panic-unwind-impl=rust/native`. This would allow those users for which the
translation is too expensive to opt-in to a better implementation for their
applications.
