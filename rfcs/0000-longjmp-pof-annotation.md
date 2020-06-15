- Feature Name: `annotation-for-safe-longjmp`
- Start Date: 2019-06-11
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)
- Project group: [FFI-unwind][project-group]

 [project-group]: https://github.com/rust-lang/project-ffi-unwind

 <!-- TODO for now, content is copied from prior drafts of the "C unwind" RFC. -->

# Motivation
[motivation]: #motivation

Additionally, there are libraries such as `rlua` that rely on `longjmp` across
Rust frames; on Windows, `longjmp` is implemented via [forced
unwinding][forced-unwinding]. The current `rustc` implementation makes it safe
to `longjmp` across Rust [POFs][POF-definition] (frames without `Drop` types),
but this is not formally specified in an RFC or by the Reference.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC specifies that, regardless of the platform or the ABI string (`"C"` or
`"C unwind"`), any platform features that may rely on forced unwinding is
defined behavior when all unwound frames are POFs

As an example:

```rust
#[cancelable]
fn foo<D: Drop>(c: bool, d: D) {
  if c {
    drop(d);
  }
  longjmp_if_true(c);
}

/// Calls `longjmp` if `c` is true; otherwise returns normally.
#[cancelable]
extern "C" fn longjmp_if_true(c: bool);
```

If a `longjmp` occurs, it can safely traverse the `foo` frame, which will be a
POF because `d` has already been dropped.

Since `longjmp_if_true` function is using the `"C"` rather than the `"C
unwind"` ABI, the optimizer may assume that it cannot unwind; on LLVM, this is
represented by the `nounwind` attribute. On most platforms, `longjmp` is not a
form of unwinding: the `foo` frame is simply discarded. On Windows, `longjmp`
is implemented as a forced unwind, which is permitted to traverse `nounwind`
frames. Since `foo` contains a `Drop` type the forced unwind will include a
call to the frame's cleanup logic, but that logic will not produce any
observable effect; in particular, `D::drop()` will not be called again. The
observable behavior should therefore be the same on all platforms.

Conversely, if, due to a bug, `longjmp` were called unconditionally, then this
code would have undefined behavior on all platforms when `c` is false, because
`foo` would not be a POF.

<!-- TODO the above only talks about UB, but we want warnings to be more
conservative: the compiler should warn in any scenario where a frame is not
guaranteed to be a POF at the time it calls a "cancelable" function. -->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Regardless of the panic runtime, ABI, or platform, the interaction of Rust
frames with C functions that deallocate frames (i.e. functions that may use
forced unwinding on specific platforms) is specified as follows:

* **When deallocating Rust [POFs][POF-definition]:** frames are safely
    deallocated; no undefined behavior
* **When deallocating Rust non-POFs:** undefined behavior

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Analysis of key design goals
[analysis-of-design-goals]: #analysis-of-design-goals

### Enable error handling with `longjmp`

This constraint is met: `longjmp` is treated the same across all platforms, and
is safe as long as only [POFs][POF-definition] are deallocated.

### Do not change the ABI of functions in the `libc` crate

This constraint is met: `libc` functions will continue to use the `"C"` ABI.
`pthread_exit` will be treated the same across all platforms, and will be safe
as long as only [POFs][POF-definition] are deallocated. 
