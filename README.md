# Project "ffi-unwind"

A [working-group project][shepherds-blog] to extend the Rust language to
support unwinding that crosses FFI boundaries.

- Shepherds:
  - [acfoltzer (Adam)](https://github.com/acfoltzer)
  - [batmanaod (Kyle)](https://github.com/batmanaod)
- Rust lang team contacts:
  - [nikomatsakis (Niko)](https://github.com/nikmoatsakis)
  - [joshtriplett (Josh)](https://github.com/joshtriplett)
- [Our chat room][zulip-room]
- [Our charter](charter.md)
- [Our project planning](project-planning.md)
- [Cross-language unwinding FAQ](faq.md)
- [Technical roadmap](roadmap/)

[shepherds-blog]: http://smallcultfollowing.com/babysteps/blog/2019/09/11/aic-shepherds-3-0/
[zulip-room]: https://rust-lang.zulipchat.com/#narrow/stream/210922-wg-ffi-unwind/topic/welcome/near/177543226

# Contributing

We do not have a formal concept of membership. Please feel free to join our
[Zulip chat room](zulip-room) or to open a GitHub Issues and/or Pull Request.

# ffi-unwind table

## Key

* :white_check_mark: -- available and well-defined on stable
* :yellow_heart: -- available on nightly
* :revolving_hearts: -- open RFC
* :speech_balloon: -- under active discussion
* :clock1: -- not yet under active discussion
* :no_entry_sign: -- out of scope for this group, no current plans to implement or specify

## Table


| Thing | linux | mac | msvc | other targets | 
| --- | --- | --- | --- | --- |
| ["C unwind" ABI] | :speech_balloon: | :speech_balloon: | :speech_balloon: | :speech_balloon: |
| [propagate Rust panic through native frame], no destructors | :clock1: | :clock1: | :clock1: |:no_entry_sign: |
| [propagate Rust panic through native frame], with destructors | :clock1: | :clock1: | :clock1: |:no_entry_sign: |
| [propagate native exception through Rust frame], no destructors | :clock1: | :clock1: | :clock1: |:no_entry_sign: |
| [propagate native exception through Rust frame], with destructors | :clock1: | :clock1: | :clock1: |:no_entry_sign: |
| catch Rust panic within native code | :no_entry_sign: | :no_entry_sign: | :no_entry_sign: | :no_entry_sign: |
| catch native exception within Rust code | :no_entry_sign: | :no_entry_sign: | :no_entry_sign:  | :no_entry_sign: |

["C unwind" ABI]: roadmap/c-unwind-abi.md
[propagate Rust panic through native frame]: roadmap/propagate-rust-panic-through-native-frame.md
[propagate native exception through Rust frame]: roadmap/propagate-rust-panic-through-native-frame.md
