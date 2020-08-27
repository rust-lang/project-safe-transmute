- Feature Name: promise_transmutable
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

[safer transmutation]: 0000-safe-transmute.md
[zerocopy]: https://crates.io/crates/zerocopy
[bytemuck]: https://crates.io/crates/bytemuck

# Summary
[summary]: #summary

This is a compiler-suported extension to [safer transmutation], adding `#[derive(PromiseTransmutable)]`, which expands to `#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]`.

# Motivation
[motivation]: #motivation

We anticipate that *most* users will merely want to promise that their types are as-stable-as-possible. To do so, [safer transmutation] provides this shorthand:
```rust
#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(C)]
pub struct Foo(pub Bar, pub Baz);
```
As a shorthand, this is still rather long. For such users, the separability of `PromiseTransmutableFrom` and `PromiseTransmutableInto` is totally irrelevant.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We propose a `derive(PromiseTransmutable)` shorthand, such that this:
```rust
#[derive(PromiseTransmutable)]
#[repr(C)]
pub struct Foo(pub Bar, pub Baz);
```
...is equivalent to this:
```rust
#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(C)]
pub struct Foo(pub Bar, pub Baz);
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

We caution *against* adding a corresponding trait or trait alias; e.g.:
```rust
trait PromiseTransmutable = PromiseTransmutableFrom + PromiseTransmutableInto;
```
The vast majority of users will *only* confront the stability declaration traits in the context of deriving them; the *only* scenario in which end-users will refer to these traits in a type-context is the rare use-case of *manually* implementing them. For such users, the separability of `PromiseTransmutableFrom` and `PromiseTransmutableInto` *is* relevant. The availability of a `PromiseTransmutable` trait or trait alias in this scenario would be a distraction, since referring to it in a type-context is almost certainly a misstep.

# Drawbacks
[drawbacks]: #drawbacks

We acknowledge that it is unusual for a `derive` macro to not implement an item of the same name, but this weirdness is outweighed by the weirdness of the alternative: providing a trait for which there is almost no good use. 
