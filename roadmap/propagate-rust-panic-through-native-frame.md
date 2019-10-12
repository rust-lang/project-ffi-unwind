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
[Rust frames, thread boundary, or app entry point]
      |
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

## Possible properties of the native frames

* The native frame has **no destructors**. An example of this might be a C function.
    * The precise meaning of "no destructors" is defined in terms of the native ABI.
    * We need to define how the Rust panic propagates across such frames.
* The native frame **has destructors** it would like to execute. These could come from C++
  RAII or similar mechanisms.
    * We need to define how the Rust panic is defined and which destructors would execute.
* The native frame has **no destructors or any affordances for
  unwinding**. An example of this might be a C function compiled with
  `-fno-exceptions` or `-fno-unwind-tables`.
    * The precise meaning of "affordances for unwinding" is defined in terms of the native ABI.

Currently, having a Rust panic "unwind" a native frame is **undefined
behavior** in all of the above cases. However, we plan to specify the
first case (no destructors) before we think about the more complex
cases (contains destructors, no affordances). We will also have to
specify this on a per-target basis, as the details will vary depending
on what exception mechanism is in use.

Another consideration is whether the the native frames include a `catch` block.
There are several possibilities here as well:

* "catch native" -- e.g. a C++ block catching a specific subset of C++ exceptions
* "catch all" -- e.g. a C++ catch without an exception argument
* "catch and release" -- e.g. a C++ "catch all" that re-`throw`s the exception

So far we do **not** have plans to specify these cases.

## Unwind-termination and related concerns

There are several possibilities for the top section of the diagram, above the
native frames:

* Rust frames - the `panic` re-enters Rust frames
  * At this point, the `panic` will be no different from a normal `panic`,
    unless and until it re-enters native frames.
  * The `panic` may caught as usual, either with `catch_panic` or by crossing a
    thread boundary.
* Thread boundary - the exception propagates all the way to a native frame that
  was invoked from another thread
* App entry point - the unwind is not caught and does not cross a thread
  boundary.
  * This should cause the application to terminate.

Again, these are all currenntly undefined.

We **do** expect to make the first and last possibilities (re-entering Rust
frames or reaching the app entry point) well-defined; their behavior should be
a natural consequence of how the `"C unwind"` ABI is defined.

We **do not** currently have plans to define the middle case, wherein a `panic`
propagates through native frames all the way to a thread boundary.

