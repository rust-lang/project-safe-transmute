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
and the [entry in the Unsafe Code Guidelines glossary][ucg-ub] for more details.

[ucg-ub]: https://rust-lang.github.io/unsafe-code-guidelines/glossary.html#undefined-behavior

Our usage of the term is generally the same as the [standard
usage](https://en.wikipedia.org/wiki/Undefined_behavior) from other
languages.

<a name="LLVM-UB"></a>

## LLVM-undefined behavior (LLVM-UB)

We use the phrase **LLVM undefined behavior** to indicate things that
are considered undefined behavior by LLVM itself. Barring bugs, the
Rust compiler should never produce LLVM IR that contains LLVM-UB
unless the behavior in question is *also* UB in Rust. Of course, there
are bugs in the Rust compiler from time to time, and hence it can
happen that we generate LLVM IR which contains LLVM-UB even if the
corresponding Rust source code is meant to be fully defined (see
e.g. [rust-lang/rust#28728]).  The main reason it is worth separating
LLVM-UB from the more general form of Rust UB is that, while both
forms of UB can cause arbitrary things to happen in your
code. However, as a practical measure, LLVM-UB is much more *likely
to* in practice.

[rust-lang/rust#28728]: https://github.com/rust-lang/rust/issues/28728

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

