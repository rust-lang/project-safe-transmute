- Feature Name: layout_traits
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

[safer transmutation]: 0000-safe-transmute.md

# Summary
[summary]: #summary

This is a **library extension** to [safer transmutation] (i.e., it does not require additional compiler support) which introduces traits that are implemented depending on the sizes and alignments of two given types:
  - `mem::AlignLtEq<Rhs>`, implemented for `Lhs` if `align_of::<Lhs>() <= align_of::<Rhs>` 
  - `mem::AlignEq<Rhs>`, implemented for `Lhs` if `align_of::<Lhs>() == align_of::<Rhs>` 
  - `mem::SizeLtEq<Rhs>`, implemented for `Lhs` if `size_of::<Lhs>() <= size_of::<Rhs>` 
  - `mem::SizeEq<Rhs>`, implemented for `Lhs` if `size_of::<Lhs>() == size_of::<Rhs>` 

# Motivation
[motivation]: #motivation

Some abstractions that depend on a type's layout are more easier thought of in terms of size or alignment constraints than transmutability. For instance:

## Pointer Bit Packing
A common bit-packing technique involves abusing the relationship between allocations and alignment. If a type is aligned to 2<sup>n</sup>, then the *n* least significant bits of pointers to that type will equal `0`. These known-zero bits can be packed with data. Since alignment cannot be currently reasoned about at the type-level, it's currently impossible to bound instantiations of a generic parameter based on minimum alignment.

Using [safer transmutation], we can require that a reference to a generic type `T` has alignment of at least a given value (e.g., `8`) by first defining a ZST with that alignment:
```rust
#[derive(PromiseTransmutableFrom)]
#[repr(align(8)]
struct Aligned8;
```
and then adding this `where` bound to our abstraction:
```rust
where
    &T: TransmuteInto<&Aligned8>
```

The intuition behind this trait bound requires a thorough understanding of transmutability. With this RFC, this bound would be more clearly expressed as:
```rust
#[derive(PromiseTransmutableFrom)]
#[repr(align(8)]
struct Aligned8;
```

## `Vec` Casting
The invariants imposed by [`Vec::from_raw_parts`](https://doc.rust-lang.org/alloc/vec/struct.Vec.html#method.from_raw_parts) stipulate that we may only convert a `Vec<T>` to `Vec<U>` if:
  - `U` is transmutable from `T`
  - `U` has the same minimum alignment as `T`
  - `U` has the same size as `T`

With this RFC, these requirements may be written directly:
```rust
fn cast<Src, Dst>(src: Vec<Src>) -> Vec<Dst>
where
    Dst: TransmuteFrom<Src>
       + AlignEq<Src>,
       + SizeEq<Src>,
{
    let (ptr, len, cap) = src.into_raw_parts();
    unsafe { Vec::from_raw_parts(ptr as *mut Dst, len, cap) }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Given `TransmuteFrom` and `TransmuteInto`, we can construct bounds that check certain properties of a type by checking if its convertible to another, contrived type. These gadgets are useful, but subtle to formulate.

## Querying Alignment
The type `[T; 0]` shares the alignment requirements of `T`, but no other layout properties. A type `&[T; 0]` will only be transmutable to `&[U; 0]`, if the minimum alignment of `T` is greater than that of `U`. We exploit this to define a trait that is implemented for `Self` if its alignment is less-than-or-equal to that of `Rhs`:
```rust
/// Implemented if `align_of::<Self>() <= align_of::<Rhs>()`
pub trait AlignLtEq<Rhs, Neglect=()>
where
    Neglect: TransmuteOptions,
{}

impl<Lhs, Rhs, Neglect> AlignLtEq<Rhs, Neglect> for Lhs
where
    Neglect: TransmuteOptions,
    for<'a> &'a [Lhs; 0]: TransmuteFrom<&'a [Rhs; 0], Neglect>
{}
```
Furthermore, if the alignment of `Self` is less-than-or-equal to `Rhs`, and the alignment of `Rhs` is less-than-or-equal to `Self`, then the alignments of `Self` and `Rhs` must be equal:
```rust
/// Implemented if `align_of::<Self>() == align_of::<Rhs>()`
pub trait AlignEq<Rhs, Neglect=()>
where
    Neglect: TransmuteOptions,
{}

impl<Lhs, Rhs, Neglect> AlignEq<Rhs, Neglect> for Lhs
where
    Neglect: TransmuteOptions,
    Lhs: AlignLtEq<Rhs, Neglect>,
    Rhs: AlignLtEq<Lhs, Neglect>,
{}
```

## Querying Size
Querying *just* the size of a type is trickier: how might we query the size without implicitly querying the alignment and validity of the bytes that contribute to that size?

We do so by constructing a contrived container type that neutralizes the alignment and validity aspects of its contents:
```rust
#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)] */
#[repr(C)]
struct Gadget<Align, Size>(pub [Align; 0], pub MaybeUninit<Size>);
```
This type will have the size of `Size`, and alignment equal to the maximum of `Align` and `Size`. `MaybeUninit<Size>` neutralizes the bit-validity qualities of `Size`.

We use this `Gadget` to define a trait that is implemented if the size of `Self` is less-than-or-equal to the size of `Rhs`:
```rust
/// Implemented if `size_of::<Self>() <= size_of::<Rhs>()`
pub trait SizeLtEq<Rhs, Neglect=()>
where
    Neglect: TransmuteOptions,
{}

impl<Lhs, Rhs, Neglect> SizeLtEq<Rhs, Neglect> for Lhs
where
    Neglect: TransmuteOptions,
    for<'a> &'a Gadget<Rhs, Lhs>: TransmuteFrom<&'a Gadget<Lhs, Rhs>, Neglect>,
{}
```
This works, because `Gadget<Rhs, Lhs>` and `Gadget<Lhs, Rhs>` will have equal alignment, and equal bit-validity (they consist solely of padding bytes). Thus, all that varies between them is their size, and reference transmutations must either preserve or reduce the size of the transmuted type.

As before, if the size of `Self` is less-than-or-equal to `Rhs`, and the size of `Rhs` is less-than-or-equal to `Self`, then the sizes of `Self` and `Rhs` must be equal:
```rust
/// Implemented if `size_of::<Self>() == size_of::<Rhs>()`
pub trait SizeEq<Rhs, Neglect=()>
where
    Neglect: TransmuteOptions,
{}

impl<Lhs, Rhs, Neglect> SizeEq<Rhs, Neglect> for Lhs
where
    Neglect: TransmuteOptions,
    Lhs: SizeLtEq<Rhs, Neglect>,
    Rhs: SizeLtEq<Lhs, Neglect>,
{}
```

# Prior art
[prior-art]: #prior-art

The [FromBits](https://github.com/joshlf/rfcs/blob/joshlf/from-bits/text/0000-from-bits.md#sizeleq-and-alignleq) pre-RFC, recommends adding two similar traits: `AlignLeq` and `SizeLeq`. This pre-RFC envisions these traits as requiring additional compiler support; our proposal formulates equivalent traits just using [safer transmutation]. 
