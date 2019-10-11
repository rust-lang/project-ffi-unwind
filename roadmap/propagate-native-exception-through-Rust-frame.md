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
      |                           ^
      | (calls)                   | (unwinding
      v                           |  goes this
[native function]                 |  way)
      |                           |
      +--- rust function panics --+
```

We can distinguish two cases:

* The Rust frame has **no in-scope destructors**.
    * We need to define precisely when a Rust function is guaranteed to have no destructors.
* The Rust frame **has destructors** it would like to execute.
    * We need to define how they interact with a native exception.

Currently, having a Rust panic "unwind" a native frame is **undefined
behavior** in both of the above cases. However, we plan to specify the
first case (no destructors) before we think about the more complex
case (contains destrucors). We will also have to specify this on a
per-target basis, as the details will vary depending on what exception
mechanism is in use.


