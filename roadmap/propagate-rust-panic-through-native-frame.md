# Propagate Rust panic through native frames

## Summary

These roadmap items refer to when a panic is raised in a Rust function
that is defined with `"C unwind"` ABI, as would happen if C or C++
code invoked the following function:

```rust
extern "C unwind" fn example() {
    panic!("Uh oh");
}
```

In this case, the stack would look like one of the two scenarios:

```
     ...
      |
[Native frame]
      |                           ^
      | (calls)                   | (unwinding
      v                           |  goes this
[Rust function `example`]         |  way)
      |                           |
      +--- rust function panics --+
```

We can distinguish two cases:

* The native frame has **no destructors**. An example of this might be a C function.
    * The precise meaning of "no destructors" is defined in terms of the native ABI.
    * We need to define how the Rust panic propagates across such frames.
* The native frame **has destructors** it would like to execute. These could come from C++
  RAII, Java finally blocks, etc.
    * We need to define how the Rust panic is defined and which destructors would execute.

Currently, having a Rust panic "unwind" a native frame is **undefined
behavior** in both of the above cases. However, we plan to specify the
first case (no destructors) before we think about the more complex
case (contains destrucors). We will also have to specify this on a
per-target basis, as the details will vary depending on what exception
mechanism is in use.

