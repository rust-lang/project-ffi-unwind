# Terminology about specifications

Language and platform specifications have several different terms used to
describe how well-defined a language feature is, i.e., how well constrained the
runtime behavior is.

The ISO C and C++ standards distinguish several degrees of specification by
assigning precise definitions to the following terms:

* Well-defined behavior
* Implementation-defined behavior
* Unspecified behavior
* Undefined behavior

Of these, only "undefined behavior" is used consistently within the Rust
project; there are _no_ guarantees about the runtime behavior of such code.

It is currently out of scope for this project to define the other terms in that
list or their relationship to the corresponding terms in other languages.
However, this project does distinguish a particular _category_ of undefined
behavior:

#### LLVM-undefined behavior (or LLVM-UB)

Cases of known LLVM-UB are a specific subset of undefined behavior in general.
Rust code with LLVM-UB will cause `rustc` to generate LLVM IR exhibiting
undefined behavior.

This is distinct from the general case of Rust undefined behavior, in which
it is unknown whether `rustc` will generate well-behaved LLVM IR.
