# Charter

### Mission statement

To extend the language to avoid undefined behavior when using control-flow
constructs that cross between language boundaries.

### Unwinding

The "MVP" for unwinding, with remaining open questions, was specified in
[RFC-2945][unwind-rfc].

### `longjmp` and `pthread_cancel`

This is under active discussion in Zulip.

[unwind-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md
