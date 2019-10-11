# Propagate native exception through Rust frame

## Summary

These roadmap items refer to when a "native exception" is raised in a
native function defined with the `"C unwind"` ABI, as might happen if
we invoked a C++ function that throws an exception.

(TODO: Investigate details of how this would be declared in any case.)

In this case, the stack would look like this:

```
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

We can distinguish two cases:

* The Rust frame has **no in-scope destructors**.
    * We need to define precisely when a Rust function is guaranteed to have no destructors.
    * This may also be useful for indicating cases where it is legal
      to `longjmp` over a frame, though that is not one of our direct
      goals.
* The Rust frame **has destructors** it would like to execute.
    * We need to define how they interact with a native exception.

Currently, having a native exception "unwind" a Rust frame is
**undefined behavior** in both of the above cases. However, we plan to
specify the first case (no destructors) before we think about the more
complex case (contains destructors). We will also have to specify this
on a per-target basis, as the details will vary depending on what
exception mechanism is in use, and what other non-exception features
(such as `longjmp`) may use the same mechanism.


