- Feature Name: byte_transmutations
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

[safer transmutation]: 0000-safe-transmute.md
[zerocopy]: https://crates.io/crates/zerocopy
[bytemuck]: https://crates.io/crates/bytemuck

# Summary
[summary]: #summary

This is a **library extension** to [safer transmutation] (i.e., it does not require additional compiler support) which introduces traits encoding common transmutation-conversions: *byte* transmutations:
  - `FromZeros`, implemented if `Self` is initializable from an equivalently-sized array of zeroed bytes.
  - `FromBytes`, implemented if `Self` is transmutable into an equivalently-sized array of initialized bytes.
  - `IntoBytes`, implemented if `Self` is transmutable from an equivalently-sized array of initialized bytes.

# Motivation
[motivation]: #motivation
Transmutations of types to-and-from equivalently-sized buffers of bytes are perhaps the most common use-case of transmutation; e.g., `FromBytes` and `AsBytes` traits of [zerocopy] form the foundation of Fuchsia's Rust networking stack. These traits can be formulated soundly and completely via [safer transmutation], but the obvious formulations aren't viable; e.g.:
```rust
// `Dst` is `FromBytes` if it can be safely transmuted *from* an
// equivalently sized array of `u8`.
unsafe impl<Dst> FromBytes for Dst
where
    Dst: TransmuteFrom<[u8; size_of::<Dst>()]>,
{}
```
At the time of writing, `size_of::<Dst>()` *cannot* appear in this context. Our proposal provides implementation of these traits that do not rely on speculative advancements in const generics.

Together, `IntoBytes` and `FromBytes` can form the basis of a [bytemuck]-like [`PlainOldData` trait](https://docs.rs/bytemuck/1.*/bytemuck/trait.Pod.html):
```rust
/// Implemented by types that are "plain old data":
pub unsafe trait PlainOldData<Neglect=()>
where
    Neglect: TransmuteOptions
{}

unsafe impl<T, Neglect> PlainOldData for T
where
    T: FromBytes<Neglect> + IntoBytes<Neglect>,
    Neglect: TransmuteOptions
{}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementations of these traits using [safer transmutation] follows:

## `FromZeros`
Indicates that a type may be transmuted from an appropriately-sized array of zeroed bytes. This trait provide a safe alternative to [`mem::zeroed()`](https://doc.rust-lang.org/core/mem/fn.zeroed.html).

```rust
pub unsafe trait FromZeros<Neglect = ()>
where
    Neglect: TransmuteOptions
{
    /// Safely initialize `Self` from zeroed bytes.
    fn zeroed() -> Self
    where
        Neglect: SafeTransmuteOptions;

    /// Unsafely initialize `Self` from zeroed bytes.
    fn unsafe_zeroed() -> Self
    where
        Neglect: TransmuteOptions;
}

#[derive(Copy, Clone, PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(u8)]
enum Zero {
    Zero = 0u8
}

unsafe impl<Dst, Neglect> FromZeros<Neglect> for Dst
where
    Dst: TransmuteFrom<[Zero; usize::MAX], Neglect>,
    Neglect: TransmuteOptions
{
    fn zeroed() -> Self
    where
        Neglect: SafeTransmuteOptions
    {
        [Zero; size_of::<Self>].transmute_into()
    }

    unsafe fn unsafe_zeroed() -> Self
    where
        Neglect: TransmuteOptions
    {
        [Zero; size_of::<Self>].unsafe_transmute_into()
    }
}
```

## `FromBytes`
Indicates that a type may be transmuted from an appropriately-sized array of bytes.
```rust
pub unsafe trait FromBytes<Neglect = ()>
where
    Neglect: TransmuteOptions
{}

unsafe impl<Dst, Neglect> FromBytes<Neglect> for Dst
where
    Dst: TransmuteFrom<[u8; usize::MAX], Neglect>,
    Neglect: TransmuteOptions
{}
```

## `IntoBytes`
Indicates that a type may be transmuted into an appropriately-sized array of bytes.
```rust
#[marker]
pub unsafe trait IntoBytes<Neglect = ()>
where
    Neglect: TransmuteOptions
{}

// covers `size_of::<Src>() >= 1`
unsafe impl<Src, Neglect> IntoBytes<Options> for Src
where
    [Src: usize::MAX]: TransmuteInto<[u8; usize::MAX], Neglect>,
    Neglect: TransmuteOptions
{}

// covers `size_of::<Src>() == 0`
unsafe impl<Src, Neglect> IntoBytes for Src
where
    Src: SizeEq<()>,
    Neglect: TransmuteOptions
{}
```
