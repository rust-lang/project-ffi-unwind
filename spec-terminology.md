# Terminology about specifications

Language and platform specifications have several different terms used
to describe how well-defined a language feature is, i.e., how well
constrained the runtime behavior is. In cases where our terminology
overlaps with that from other communities, we try to remain generally
compatible.

<a name="UB"></a>

## Undefined Behavior

As is typical within the Rust community we use the phrase **undefined
behavior** to refer to illegal program actions that can result in
arbitrary results. In short, "undefined behavior" is always a bug and
never something you should do. See the [Rust
reference](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)
for more details.

Our usage of the term is generally the same as the [standard
usage](https://en.wikipedia.org/wiki/Undefined_behavior) from other
languages.

<a name="LLVM-UB"></a>

### LLVM-undefined behavior (LLVM-UB)

As a special case of undefined behavior, we use the phrase **LLVM
undefined behavior** to indicate things that are considered undefined
behavior by LLVM itself (as opposed to by the Rust compiler). There is
no theoretical difference between UB and LLVM-UB -- both can cause
arbitrary things to happen in your code. However, as a practical
measure, LLVM-UB is worth distinguishing because it is much more
*likely to* in practice.

<a name="unspecified"></a>

## Unspecified behavior

We use the term "unspecified behavior" to refer to behavior that may
vary across Rust releases, depending on what options are given to the
compiler, or even -- in extreme cases -- across executions of the Rust
compiler. However, unlike undefined behavior, the resulting execution
is not completely undefined, and it must typically fall within some
range of possibilities. Often, we will not specify precisely *how*
something is implemented, but rather the patterns that must work.

An example of "unspecified behavior" is the [layout for structs with
no declared `#[repr]` attribute][ucg-struct].  This layout can and
does change across Rust releases -- but of course within a given
compilation, a struct must have *some* layout. Moreover, we guarantee
that programs can (for example) use `sizeof` to determine the size of
that layout, or access fields using Rust syntax like `foo.bar`. This
requires the layout to be communicated in some fashion but doesn't
specify how that is done.

[ucg-struct]: https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/layout/structs-and-tuples.md

Our usage of the term is generally the same as the [standard
usage](https://en.wikipedia.org/wiki/Unspecified_behavior) from other
languages.

<a name="TBD"></a>

## To Be Defined Behavior (TBD)

We refer to some behavior as **to be defined** to indicate that --
while it is currently unspecified -- we *intend* to define that
behavior at some point as part of this project group (though plans can
change). This helps to define the scope of the group, but it also
indicates behavior that you would be able to rely upon in the future.
Note that TBD behavior is **still unspecified** until a formal
decision is made, though, so if you rely on it today, your code may
stop working or work differently under future releases of Rust (even
if it compiles on the stable compiler).
