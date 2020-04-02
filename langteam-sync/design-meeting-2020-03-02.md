This was the originally scheduled date for the design meeting, but due to a
miscommunication, the lang team was not in attendance. The actual meeting
occurred on [March 16th](design-meeting-2020-03-16.md).

Participants: Amanieu and Kyle Strand
Notes copied from https://hackmd.io/rG_5ksyCTuKsjks5cHONZQ

# FFI-unwind design meeting 3/2

## Prefer proposal 2 to proposal 1

Preserving `longjmp` behavior on Windows regardless of `panic=abort` is important.

Distinguishing between forced/unforced foreign exceptions is technically feasible (this was the main concern about implementing proposal 2)

  - On libunwind, check the `_US_FORCE_UNWIND` flag.
  - On SEH, do the same as `catch (...)` which doesn't catch `longjmp`.

## Non-ABI attributes could be added to proposal 3 for parity of expressiveness w/ proposal 2

  `#[nounwind]` - a la C++
  `#[abort_if_panic_equals_abort]` (name TBD) - a way to mark external functions such that even with `panic=abort`, they will be invoked with a shim that has landing pads to abort when a non-forced exception is encountered (this is the behavior of `"C unwind"` in proposal 2)

### What about function pointers?

  Proposal 2 provides a way to limit code size by marking *function pointers* as "nounwind" (i.e. `extern "C"`). For Proposal 3, to get the same optimization, the function pointer would need to be given an attribute, which currently isn't part of the language; also, this annotated type would need to be a different _type_.

  ```rust
  // Proposal 3, after adding function-pointer-annotations to the language

  extern "C" {
#[nounwind] fn marked_nounwind();
#[abort_if_panic_equals_abort] fn marked_abort();
    fn default_abi();
  }

fn calls_nounwind(f: #[nounwind] fn()) {
  f()
}

fn main () {
  calls_nounwind(marked_nounwind);     // UB if unwinds
  calls_nounwind(unmarked);            // compile error :D
  default_abi();                       // UB if unwinds under -C panic=abort
  marked_nounwind();                   // UB if unwinds
  marked_abort();                      // aborts if unwinds under -C panic=abort
}
```

```rust
// Proposal 2, no function-pointer-annotations

extern "C unwind" {
  fn marked_unwind();
}

extern "C" {
  fn default_abi();
}

fn calls_nounwind(f: extern "C" fn()) {
  f()    // Aborts
}

fn main () {
  calls_nounwind(default_abi);      // UB if unwinds
  calls_nounwind(marked_unwind);    // compile error :D
  marked_unwind();                  // aborts if unwinds under -C panic=abort
  default_abi();                    // UB if unwinds
}
```

### Do we need "unwind" variants for other ABIs? (e.g. "system")

"system" is only used on x86 Windows, probably not worth supporting `"system unwind"`. Same with other ABIs.
