# Miscellaneous terminology

## Native vs foreign

[Zulip conversation][zulip]

We will avoid the use of the term "native (stack) frames", since it is
ambiguous. Instead, we will refer to "Rust frames" versus "foreign frames",
where "foreign" will mean stack frames from languages other than Rust.

We will still use the word "native" to describe the exception _mechanism_,
which is distinct from "foreign" exception _objects_ (which means exception
objects created in other languages).

[zulip]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/Native.20frame.20definition/near/178713063
