- Feature Name: castfrom
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

[safer transmutation]: 0000-safe-transmute.md
[zerocopy]: https://crates.io/crates/zerocopy
[packet]: https://fuchsia-docs.firebaseapp.com/rust/packet

# Summary
[summary]: #summary

This is a **library extension** to [safer transmutation] (i.e., it does not require additional compiler support) which introduces traits for transmute-like conversions between container-types like `Vec` and slices.

# Motivation
[motivation]: #motivation

The [safer transmutation] API makes it possible to define safe mechanisms for transmuting the contents of slices and `Vec`s. These conversions are not, themselves, sound *transmutations*: neither the layout of slices nor `Vec`s are well-defined. However, they each provide `from_raw_parts` constructors that can be used to effectively transmute their contents.


A slice cast is an operation that consumes `&'i [Src]` and produces `&'o [Dst]`. This conversion has both static and dynamic components. The length of a `&'i [Src]` is a dynamic quality of values of that type. If ths static sizes of `Src` and `Dst` differ, the length of `&'o [Dst]` will need to differ from that of `&'i [Src]` accordingly. The static component of this conversion is whether the layouts of `Src` and `Dst` even permit such a conversion---we can use `TransmuteFrom` to answer this: *Could a reference to a maximally-long array of `Src` be transmuted to an array of just one `Dst`?*

Concretely:
```rust
fn cast<'i, 'o, Src, Dst>(src: &'i [Src]) -> &'o [Dst]
where
    &'o [Dst; 1]: TransmuteFrom<&'i [Src; usize::MAX]>
{
    let len = size_of_val(src).checked_div(size_of::<Dst>()).unwrap_or(0);
    unsafe { slice::from_raw_parts(src.as_ptr() as *const Dst, len) }
}
```

The invariants imposed by [`Vec::from_raw_parts`](https://doc.rust-lang.org/alloc/vec/struct.Vec.html#method.from_raw_parts) are far stricter than those imposed by [`slice::from_raw_parts`](https://doc.rust-lang.org/core/slice/fn.from_raw_parts.html): we may only convert a `Vec<T>` to `Vec<U>` if:
  - `U` is transmutable from `T`
  - `U` has the same size as `T`
  - `U` has the same static alignment as `T`

Concretely:
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

The above `cast` functions have a major ergonomic drawback: they require callers to reiterate their `where` bounds. The bounds that determine the castability of a type is more ergonomically encapsulated with a *single* trait bound whose name clearly communicates that casting is possible.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Mirroring the design of `TransmuteFrom`, this *could* take the form of a `CastFrom` trait:
```rust
pub mod cast {

    /// Instantiate `Self` from a value of type `Src`.
    ///
    /// The reciprocal of [CastInto].
    pub trait CastFrom<Src: ?Sized, Neglect=()>
    where
        Neglect: options::CastOptions,
    {
        /// Instantiate `Self` by casting a value of type `Src`, safely.
        fn cast_from(src: Src) -> Self
        where
            Src: Sized,
            Self: Sized,
            Neglect: options::SafeCastOptions
        {
            unsafe { CastFrom::<_,Neglect>::unsafe_cast_from(src) }
        }

        /// Instantiate `Self` by casting a value of type `Src`, potentially safely.
        unsafe fn unsafe_cast_from(src: Src) -> Self
        where
            Src: Sized,
            Self: Sized,
            Neglect: options::CastOptions;
    }

    /// Options for casting.
    pub mod options {

        /// The super-trait of all *safe* casting options.
        #[marker] pub trait SafeCastOptions: CastOptions {}

        /// The super-trait of all casting options.
        #[marker] pub trait CastOptions {}

        impl SafeCastOptions for () {}
        impl CastOptions for () {}

    }
}
```
...for which implementations on `Vec` and slices will be provided.

Using this foundation, it is trivial to define the abstractions of the [zerocopy] and [packet] crates; e.g.:
```rust
impl<'a> &'a [u8]
{
    /// Read `&T` off the front of `self`, and shrink the underlying slice.
    /// Analogous to:
    ///   - https://fuchsia-docs.firebaseapp.com/rust/packet/trait.BufferView.html#method.peek_obj_front
    ///   - https://fuchsia-docs.firebaseapp.com/rust/packet/trait.BufferView.html#method.take_obj_front
    fn take_front<'t, T>(&'a mut self) -> Option<&'a T>
    where
        Self: CastInto<&'a [T]>,
    {
        let idx = size_of::<T>();
        let (parsable, remainder) = self.split_at(idx);
        *self = remainder;
        parsable.cast_into().first()
    }

    /// Read `&T` off the back of `self`, and shrink the underlying slice.
    /// Analogous to:
    ///   - https://fuchsia-docs.firebaseapp.com/rust/packet/trait.BufferView.html#method.peek_obj_back
    ///   - https://fuchsia-docs.firebaseapp.com/rust/packet/trait.BufferView.html#method.take_obj_back
    fn take_back<'t, T>(&'a mut self) -> Option<&'a T>
    where
      Self: CastInto<&'a [T]>,
    {
        let idx = self.len().saturating_sub(size_of::<T>());
        let (remainder, parsable) = self.split_at(idx);
        *self = remainder;
        parsable.cast_into().first()
    }
}
```

To parse a `UdpPacket` given a byte slice,  we split the slice into a slice containing first 8 bytes, and the remainder. We then cast the first slice into a reference to a `UdpHeader`. A `UdpPacket`, then, consists of the reference to the `UdpHeader`, and the remainder slice. Concretely:
```rust
pub struct UdpPacket<'a> {
    hdr: &'a UdpHeader,
    body: &'a [u8],
}

#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(C)]
struct UdpHeader {
    src_port: [u8; 2],
    dst_port: [u8; 2],
    length:   [u8; 2],
    checksum: [u8; 2],
}

impl<'a> UdpPacket<'a> {
    pub fn parse(mut bytes: &'a [u8]) -> Option<UdpPacket> {
        Some(UdpPacket { hdr: {bytes.take_front()?}, body: bytes })
    }
}
```
(While this example is simple, the technique can be [expanded](https://fuchsia-docs.firebaseapp.com/rust/packet_formats/) to arbitrarily complex structures.)


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This `CastFrom` trait might then be implemented like so for `Vec`:
```rust
/// A contiguous growable array type with heap-allocated contents, `Vec<T>`.
pub mod vec {
    use core::convert::{
        transmute::{
            TransmuteFrom,
            options::{SafeTransmuteOptions, TransmuteOptions, NeglectAlignment},
        },
        cast::{
            CastFrom,
            options::{
                SafeCastOptions,
                CastOptions,
            },
        },
    };

    /// Safe options for casting `Vec<T>` to `Vec<U>`.
    pub trait SafeVecCastOptions
        : SafeCastOptions
        + SafeTransmuteOptions
        + VecCastOptions
    {}

    /// Options for casting `Vec<T>` to `Vec<U>`.
    pub trait VecCastOptions
        : TransmuteOptions
        + CastOptions
    {}

    impl<Neglect: SafeVecCastOptions> SafeCastOptions for Neglect {}
    impl<Neglect: SafeTransmuteOptions> SafeVecCastOptions for Neglect {}

    impl<Neglect: VecCastOptions> CastOptions for Neglect {}
    impl<Neglect: TransmuteOptions> VecCastOptions for Neglect {}


    use core::mem::{MaybeUninit, SizeEq, AlignEq};

    impl<Src, Dst, Neglect> CastFrom<Vec<Src>, Neglect> for Vec<Dst>
    where
        Neglect: VecCastOptions,
        Dst: TransmuteFrom<Src, Neglect>
           + AlignEq<Src, Neglect>
           + SizeEq<Src, Neglect>,
    {
        #[doc(hidden)]
        #[inline(always)]
        unsafe fn unsafe_cast_from(src: Vec<Src>) -> Vec<Dst>
        {
            let (ptr, len, cap) = src.into_raw_parts();
            Vec::from_raw_parts(ptr as *mut Dst, len, cap)
        }
    }
}
```

...and like so for slices:
```rust
pub mod slice {
    use core::convert::{
        transmute::{
            TransmuteFrom,
            options::{SafeTransmuteOptions, TransmuteOptions},
        },
        cast::{
            CastFrom,
            options::{
                SafeCastOptions,
                CastOptions,
            },
        },
    };

    use core::{
        mem::{size_of, size_of_val},
        slice
    };

    /// *Safe* options for casting **slices**.
    pub trait SafeSliceCastOptions
        : SafeCastOptions
        + SafeTransmuteOptions
        + SliceCastOptions
    {}

    /// Options for casting **slices**.
    pub trait SliceCastOptions
        : CastOptions
        + TransmuteOptions
    {}

    impl<Neglect: SafeSliceCastOptions> SafeCastOptions for Neglect {}
    impl<Neglect: SafeTransmuteOptions> SafeSliceCastOptions for Neglect {}

    impl<Neglect: SliceCastOptions> CastOptions for Neglect {}
    impl<Neglect: TransmuteOptions> SliceCastOptions for Neglect {}


    impl<'i, 'o, Src, Dst, Neglect> CastFrom<&'i [Src], Neglect> for &'o [Dst]
    where
        Neglect: SliceCastOptions,
        &'o [Dst; 1]: TransmuteFrom<&'i [Src; usize::MAX], Neglect>
    {
        unsafe fn unsafe_cast_from(src: &'i [Src]) -> &'o [Dst]
        {
            let len = size_of_val(src).checked_div(size_of::<Dst>()).unwrap_or(0);
            unsafe { slice::from_raw_parts(src.as_ptr() as *const Dst, len) }
        }
    }

    impl<'i, 'o, Src, Dst, Neglect> CastFrom<&'i mut [Src], Neglect> for &'o mut [Dst]
    where
        Neglect: SliceCastOptions,
        &'o mut [Dst; 1]: TransmuteFrom<&'i mut [Src; usize::MAX], Neglect>
    {
        unsafe fn unsafe_cast_from(src: &'i mut [Src]) -> &'o mut [Dst]
        {
            let len = size_of_val(src).checked_div(size_of::<Dst>()).unwrap_or(0);
            unsafe { slice::from_raw_parts_mut(src.as_ptr() as *mut Dst, len) }
        }
    }

    impl<'i, 'o, Src, Dst, Neglect> CastFrom<&'i mut [Src], Neglect> for &'o [Dst]
    where
    {
        unsafe fn unsafe_cast_from(src: &'i mut [Src]) -> &'o [Dst]
        {
            let len = size_of_val(src).checked_div(size_of::<Dst>()).unwrap_or(0);
            unsafe { slice::from_raw_parts(src.as_ptr() as *const Dst, len) }
        }
    }
}
```
