# Scope of project, by feature and platform

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

["C unwind" ABI]: planning/roadmap/c-unwind-abi.md
[propagate Rust panic through native frame]: planning/roadmap/propagate-rust-panic-through-native-frame.md
[propagate native exception through Rust frame]: planning/roadmap/propagate-rust-panic-through-native-frame.md
