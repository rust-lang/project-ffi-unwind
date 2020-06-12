- Feature Name: `annotation-for-safe-longjmp`
- Start Date: 2019-06-11
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)
- Project group: [FFI-unwind][project-group]

 [project-group]: https://github.com/rust-lang/project-ffi-unwind

# Motivation
[motivation]: #motivation

Additionally, there are libraries such as `rlua` that rely on `longjmp` across
Rust frames; on Windows, `longjmp` is implemented via [forced
unwinding][forced-unwinding]. The current `rustc` implementation makes it safe
to `longjmp` across Rust [POFs][POF-definition] (frames without `Drop` types),
but this is not formally specified in an RFC or by the Reference.

