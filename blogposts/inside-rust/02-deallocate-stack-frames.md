<!-- title TBD -->
# Rust & the case of the disappearing stack frames

-- Kyle Strand on behalf of the FFI-unwind project group --

Now that the [FFI-unwind Project Group][proj-group-gh] has merged [an
RFC][c-unwind-rfc] specifying the `"C unwind"` ABI and removing some instances
of undefined behavior in the `"C"` ABI, we are ready to establish new goals for
the group.

Our most important task, of course, is to implement the newly-specified
behavior. This work is underway and can be followed [here][c-unwind-pr].
<!-- TODO: Katie: would you like a personal call-out here? -->

The requirements of our current charter, and the [RFC creating the
group][proj-group-rfc], are effectively fulfilled by the specification of `"C
unwind"`, so one option is to simply wind down the project group. While
drafting the `"C unwind"` RFC, however, we discovered that the existing
guarantees around `longjmp` and similar functions could be improved. Although
this is not strictly related to unwinding[1], they are closesly related: they are
both "non-local" control-flow mechanisms that prevent functions from
returning normally. Because one of the goals of the Rust project is for Rust to
interoperate with existing C-like languages, and these control-flow mechanisms
are widely used in practice, we believe that Rust must have some level of
support for them.

This blog post will explain the problem space. If you're interested in helping
specify this behavior, please come join us in [our Zulip
stream][proj-group-zulip]!

<!-- XXX make this a real footnote somehow? Also, clean up this explanation... -->
[1] As mentioned in the RFC, on Windows, `longjmp` actually *is* an unwinding
operation. On other platforms, however, `longjmp` is unrelated to unwinding.

## `longjmp` and its ilk

Above, I mentioned `longjmp` and "similar functions". Within the context of the
`"C unwind"` PR, this referred to functions that have different implementations
on different platforms, and which, on *some* platforms, rely on [forced
unwinding][forced-unwinding]. In our next specification effort, however, we
would like to ignore the connection to unwinding entirely, and define a class
of functions with the following characteristic:

> a function that causes a "jump" in control flow by deallocating some number of
> stack frames without performing any additional "clean-up" such as running
> destructors

This is the class of functions we would like to address. The other primary
example is `pthread_exit`. As part of our specification, we would like to
create a name for this type of function, but we have not settled on one yet;
for now, we are referring to them as "cancelable", "`longjmp`-like", or
"stack-deallocating" functions.

## Our constraints

Taking a step back, we have two mandatory constraints on our design:

* There must be sound way to call `libc` functions that may `pthread_cancel`.
* There must be a sound way for Rust code to invoke C code that may `longjmp`
  over Rust frames.

In addition, we would like to adhere to several design principles:

* The specified behavior can't be target-platform-specific; in other words, our
  specification of Rust's interaction with `longjmp` should not depend on
  whether `longjmp` deallocates frames or initiates a forced-unwind.
  Optimizations, however, *can* be target-platform-specific.
* There should be no difference in the specified behavior of frame-deallocation
  performed by `longjmp` versus that performed by `pthread_cancel`.
* We will only permit canceling POFs.

## POFs and stack-deallocating functions

The `"C unwind"` RFC introduced a new concept designed to help us deal with
force-unwinding or stack-deallocating functions: the POF, or "Plain Old Frame".
<!-- TODO: link to section, short explanation here -->

From the definition, it should be clear that it is dangerous to call a
stack-deallocating function in a context that could destroy a non-POF stack
frame. A simple specification for Rust's interaction with stack-deallocating
functions, then, could be that they are safe to call as long as only POFs are
deallocated. This would make Rust's guarantees for `longjmp` essentially the
same as C++'s.

For now, however, we are considering POFs to be "necessary but not sufficient."
We believe that a more restrictive specification may provide the following
advantages:

* more opportunities for helpful compiler warnings or errors to prevent misuse
  of stack-deallocation functions
* semantic tracatbility: we can make reliance on stack-frame-deallocation
  visible for all functions involved
* increased optimization potential when cleanup is "guaranteed" (i.e., the
  compiler may turn a POF into a non-POF if it knows that this is safe and that
  the newly inserted cleanup operation is necessary for an optimization)

## Annotating POFs

Our current plan is to introduce a new annotation for frames that are intended
to be safe to deallocate. These functions, of course, must be POFs, and they
also cannot invoke any functions without this annotation. This makes
stack-deallocation "transitive", just like `async`: functions without this
annotation either must not invoke any annotated functions or must guarantee
that they will cause the stack-deallocation to terminate (for instance, a
non-POF, non-annotated function may call `setjmp`).
<!-- TODO improve explanation -->

The name of the annotation should be based on the terminology used to refer to
functions that are safe to deallocate. Because this terminology is not
finalized, we do not yet have a name for the annotation.

## Join us!

If you would like to help us create this specification and write an RFC for it,
please join us in [zulip][proj-group-zulip]!

[proj-group-gh]: https://github.com/rust-lang/project-ffi-unwind
[proj-group-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/2797-project-ffi-unwind.md
[proj-group-zulip]: https://rust-lang.zulipchat.com/#narrow/stream/210922-project-ffi-unwind/topic/welcome.2C.20redux/near/216807277
[c-unwind-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/2945-c-unwind-abi.md
[c-unwind-pr]: https://github.com/rust-lang/rust/pull/76570
[forced-unwinding]: XXX
