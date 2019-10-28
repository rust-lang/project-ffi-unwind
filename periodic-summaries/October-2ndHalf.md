## `catch_unwind` and foreign exceptions

[Zulip][catch-foreign-exceptions]

(TODO: add this to "resolved concerns")

gnzlbg: "we kind of need to be able to do this"; link to [Rust bug about
`ForceUnwind`][rustc-65441]

acfoltzer & Amanieu: should never catch foreign exceptions

acfoltzer: "major red flags...constrain Rust panic mechanism"

JoshT: not "the original problem the working group set out to solve"

Conclusion: leave undefined (_not_ TBD) for now; also, gnzlbg's issue was
addressed by a PR by Amanieu (see below)

## Unspecified behavior compile error

[Zulip][unspec-behavior-compile]

Code that does not compile is by definition not exhibiting "unspecified
behavior."

However, code may have unspecified on some platforms but fail to compile on
others (though it may compile in the future).

## Amanieu PR for foreign exceptions

Amanieu wrote [a PR][foreign-exceptions-pr] to ensure that foreign exceptions
can propagate through Rust frames:

> * Drop code will be executed as the exception unwinds through the stack, as
>   with a Rust panic.
> * `catch_unwind` will not catch the exception, instead the exception will
>   silently continue unwinding past it.

## "native" vs "foreign"

[Zulip][native-frame-def]

Kyle S: only use "native" to refer to exception mechanism; use "foreign" when
referring to non-Rust frames

## letting `extern "C"` unwind

[Zulip][let-extern-c-unwind]

[Shared HackMD write-up][unwind-by-default-hackmd]

Amanieu provided a detailed analysis of the behavior of cross-language
unwinding in various circumstances.

We believe that there is no undefined behavior at the LLVM level even _without_
`-fexceptions`, though we need to confirm w/ LLVM owners that it is not
automatically UB to unwind through C code compiled without `-fexceptions`

[Concern about `pthread_exit`][pthread_exit]

[Centril objects][centril-unwind-objectsions]:
* possible breaking changes
* de-facto constraint on `panic` implementation
* all implementation-defined behavior is, in practice, "defined for the whole
  language" since we only have one implementation
* Need relevant LLVM guarantees to be stated in LangRef

Josh T: unwind by default sounds reasonable

Kyle: Let's proceed w/ `extern "C unwind"`; nothing prevents us from landing
`extern "C unwind"` and then later making `extern "C"` behave the same way,
unless we make the abort-on-unwind behavior "well defined" (and therefore
stable forever, at least until the next edition)

## Noexcept-like feature

[Zulip][noexcept-feature]

gnzlbg: attribute is a bigger lang change than new ABI would be

Amanieu: not sure `noexcept` would be as useful in Rust as in C++

[Kyle][noexcept-not-now]: I think `noexcept` probably falls under project group
scope, but it is not a high priority and doesn't block other work

## Participant status updates

Is this section useful?

* acfoltzer: really busy through mid-November
* Alex Crichton: not caught up on threads but open to being pinged for
  questions
* Niko: active in conversations, etc but not entirely caught up on channels
* Amanieu: didn't know about project group before, but participating now

## Kyle propsoed action items

[Zulip][kyle-action-items]

"more or less in parallel"

* RFC for announcing project group (*no* lang changes proposed)
* continue to design `extern "C unwind"`
* stabilize the abort-on-unwind-through-`extern "C"` logic
* possibly, design some explicit opt-in syntax for the abort-on-unwind feature

## Misc

* [RustConf talk on interacting w/ C++ exceptions][katharina-talk]
* [Caution about scope creep based on Itanium ABI][scope-itanium]

[rustc-65441]: https://github.com/rust-lang/rust/issues/65441
[catch-foreign-exceptions]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/meeting.20time/near/178406886
[katharina-talk]: https://youtu.be/kdxAkCpKkbY
[unspec-behavior-compile]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/meeting.20time/near/178414227
[scope-itanium]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/WG.20scope/near/178415057
[foreign-exceptions-pr]: https://github.com/rust-lang/rust/pull/65646
[native-frame-def]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/Native.20frame.20definition/near/178713063
[let-extern-c-unwind]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/Allow.20unwinding.20from.20extern.20.22C.22.20by.20default/near/178725735
[pthread_exit]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/Allow.20unwinding.20from.20extern.20.22C.22.20by.20default/near/178758001
[unwind-by-default-hackmd]: https://hackmd.io/ymsEL6OpR6OSMoFr1As1rw
[noexcept-feature]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/noexcept-like.20feature/near/178839070
[noexcept-not-now]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/noexcept-like.20feature/near/178880684
[centril-unwind-objectsions]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/Allow.20unwinding.20from.20extern.20.22C.22.20by.20default/near/178900804
[kyle-action-items]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/lang.20team.20sync/near/179002351
