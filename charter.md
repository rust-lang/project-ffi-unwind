# Charter

To extend the language to avoid undefined behavior when using control-flow
constructs that cross between language boundaries.

Questions:

* stabilization time frame
    * talk to package vendors to understand their constraints
* stabilization "MVP"
    * add `extern "C unwind"` ABI
    * precise unwinding mechanism is not yet specified but is intended to match the "native unwinding" mechanism
    * informally, we will not emit code that intentionally interferes with unwinding
    * note that this means that there is no stable way to write code that interacts with "C unwind" ABIs that actually unwind
        * call to foreign function with that ABI and it unwinds
        * Rust function defined with that ABI unwinds
        * behavior of `catch_unwind` if a native exception should propagate
* can we stabilize a notion of "idempotent" frames -- could be useful
  for signal handlers / longjmp. Would apply potentially to native
  frames too. Think of "C code compiled with `-fexceptions`".
* is setjmp/longjmp in scope? Interacts because of SEH.
