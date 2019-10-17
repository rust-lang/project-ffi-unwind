# TBD behavior on stable

## Three categories of behavior

* Undefined Behavior -- always a bug. Conceptually, the "miri virtual machine" panics here. This is the same definition [in use by the unsafe code guidelines](https://rust-lang.github.io/unsafe-code-guidelines/glossary.html#undefined-behavior) group
* Unspecified Behavior or Details -- the "miri virtual machine" does not panic, but we don't say what happens. As you point out, there are a number of *reasons* we might not have said what happens yet:
    * We do not expect to *ever* specify this, because we want the freedom to change it.
        * Example: Rust struct layout (without a repr attribute) -- not strictly a *behavior*, of course
        * Example: whether some local variable is materialized on the stack.
    * We *may* specify this at some point in the future, but there are no plans to do so.
        * Example: Rust ABI compatibility
        * Example: What symbols get exported by a DLL
    * We wish to specify this behavior in the near future, as part of the FFI-unwind project
        * This is what I am calling **to be determined behavior**, but this is not a term that would appear in the Rust reference. In that context, we would describe such behavior as unspecified, but link to the ffi-unwind project as a place where users can learn more.
        * Examples:
            * Details of how a Rust panic presents itself in "C unwind" ABI on msvc

## TBD as an "project-local planning measure"

TBD is perhaps *best* understood as an "internal planning" device. The ffi-unwind group calls such behavior TBD if the ffi-unwind group expects to specify it at some point.

## Controversy: should TBD behavior hit stable?

* Unspecified behavior hits stable all the time, that's normal
* But the more we can minimize unspecified behavior on stable, the better, because it leaves room for "de facto" dependencies to form
* If we truly wish to avoid TBD behavior, it will be hard to stabilize "C unwind" for any platform without fully specifying all details.
    * Specifically, I think we agree that "unwinding native frames with no destrutors" are an easy case that don't require us to specify as many things -- but those frames are out of our control, so we once we permit a panic to propagate through a "C unwind" boundary, we can't really restrict what the native frames might do
* The whole problem here is that there exists code using unwinding today (on stable, I believe)
    * We are going to make that code migrate to "C unwind"
    * If they could migrate to nightly, they would've done so already and used `#[unwind(allows)]`
    * What if *in their specific case* we have decided enough for their behavior to be considered stable (e.g., they don't have destructors that might execute)
* We are in active communication with (many of) the libary authors in question, and the ffi-unwind group is actively working on specifying those TBD details.

## Core tradeoff

We are fundamentally weighing two things against one another

* the risk of de facto dependencies (which favors nightly)
* preventing libraries from using the feature on stable (which favors stable)

## Resolution

Ordinarily we definitely lean towards preventing de facto dependencies, and that is the correct call. In this case, de facto dependencies have developed (on the "C" ABI, presently), and we are attempting to get the semantics we want without breaking those dependencies (although they do require changes). **Therefore, it makes sense for us to permit "C unwind" on stable, with the explicit caveat that some details are unspecified (but we are working on it).**

