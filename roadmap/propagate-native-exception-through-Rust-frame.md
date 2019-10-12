# Propagate native exception through Rust frame

## Summary

These roadmap items refer to when a "native exception" is raised in a
native function defined with the `"C unwind"` ABI, as might happen if
we invoked a C++ function that throws an exception.

(TODO: Investigate details of how this would be declared in any case.)

In this case, the stack would look like this:

```
[Native frames, thread boundary, or app entry point]
      |
     ...
      |
[Rust frame]
      |                            ^
      | (calls)                    | (unwinding
      v                            |  goes this
[native function]                  |  way)
      |                            |
      +--- native function throws -+
```

## Possible properties of the Rust frames

* The Rust frame has **no in-scope destructors**.
  * We do not currently have a language or tooling mechanism for guaranteeing
    that Rust function is guaranteed to have no destructors. There is an
    [existing suggestion][centril-effects] for a language feature to
    provide such a guarantee. This may also be useful for indicating cases
    where it is legal to `longjmp` over a frame, though that is not one of
    our direct goals. See [the FAQ][FAQ-longjmp].
* The Rust frame **has destructors** it would like to execute.
  * We need to define how they interact with a native exception.

Currently, having a native exception "unwind" a Rust frame is
**undefined behavior** in both of the above cases. However, we plan to
specify the first case (no destructors) before we think about the more
complex case (contains destructors). We will also have to specify this
on a per-target basis, as the details will vary depending on what
exception mechanism is in use, and what other non-exception features
(such as `longjmp`) may use the same mechanism.

[centril-effects]: https://github.com/Centril/rfc-effects/issues/11
[FAQ-longjmp]: faq.md#how-does-cross-language-unwinding-differ-from-cross-language-setjmplongjmp

## Unwind-termination and related concerns

There are several possibilities for the top section of the diagram, above the
Rust frames:

* Native frames - the native unwind re-enters native frames
  * The native code runtime should be able to treat this as a normal exception
    at this point, as though Rust had never been involved.
* Thread boundary - the exception propagates all the way to a Rust frame that
  was invoked from another thread
* App entry point - the unwind is not caught and does not cross a thread
  boundary

Again, these are all currenntly undefined.

We **do** expect to make the first and last possibilities (re-entering native
frames or reaching the app entry point) well-defined; their behavior should be
a natural consequence of how the `"C unwind"` ABI is defined.

We **do not** currently have plans to define the middle case, wherein a native
unwind propagates through Rust frames all the way to a thread boundary.
