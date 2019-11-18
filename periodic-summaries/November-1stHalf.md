# Zulip topic "collections", 1st half of Nov.

## Reconsidering `"C unwind"`

This is by far the most important and most discussed topic from this period.

* [First HackMD summary & conversation](https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/.22C.20unwind.22.20vs.20.22C.20noexcept.22/near/179273421)
* [More followup to first HackMD](https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/ffi-unwind.20.E2.80.93.20meaning.20of.20C/near/180220435)
* [`extern "C"` introduces an "accidental" `noexcept` feature](https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/noexcept-like.20feature/near/178839028)
* [On whether FFI should ever change semantic behavior of callee](https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/FFI.20.22change.20behavior.20or.20protect.20the.20caller.22/near/180783324)

## Recommending immediate action to lang team on the existing soundness bug w/ `noexcept` in non-`unsafe` code

[three approaches; link to GitHub PR; discussion](https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/.22fixing.20the.20soundness.20bug.22/near/180788544)

## Catching foreign exceptions

[`catch_unwind` won't allow valid *access* to exception objects from user code](https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/Catching.20foreign.20exceptions/near/179248202)

## Measurements of binary-size impact

* [Some data](https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/measuring.20code.20size/near/180209406)
* [A branch to measure impact of safeguards for `panic = abort`](https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/implementing.20.22abort.20on.20FFI.20call.22/near/180636576)
