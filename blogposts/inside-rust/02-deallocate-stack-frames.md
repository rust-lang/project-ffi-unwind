<!-- title TBD -->
# Rust & the case of the disappearing stack frames

-- Kyle Strand on behalf of the FFI-unwind project group --

<!--

* MUSTs:
  * must be sound way to call `libc` functions that may `pthread_cancel`
  * must be a sound way for `rlua` &c to `longjmp` over Rust frames
* WANTs:
  * specified behavior can't be target-platform-specific
  * optimizations *can* be target-platform-specific
  * no behavior-spec difference between `longjmp` and `pthread_cancel`
  * only permit canceling POFs
* POFs - necessary but not sufficient
  * semantic tracatbility - `longjmp` reliance is visible for all functions
    involved
  * optimization potential when cleanup is "guaranteed"
  * possibilities for compiler warnings & errors
* annotation - need to bikeshed name

-->
