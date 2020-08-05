# Safe Transmute RFC

- Feature Name: `safe_transmute`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)


# Summary
[summary]: #summary

We propose traits, namely `TransmuteInto` and `TransmuteFrom`, that are implemented *automatically* for combinations of types that may be safely transmuted. In other words, this RFC makes safe transmutation *as easy as 1..., 2..., `repr(C)`!*
```rust
use core::transmute::{
    TransmuteInto,
    stability::{PromiseTransmutableInto, PromiseTransmutableFrom},
};

#[derive(PromiseTransmutableInto, PromiseTransmutableFrom)] // declare `Foo` to be *stably* transmutable
#[repr(C)]
pub struct Foo(pub u8, pub u16);
//                    ^ there's a padding byte here, between these fields

// Transmute fearlessly!
let _ : Foo = 64u32.transmute_into(); // Alchemy Achieved!
//                  ^^^^^^^^^^^^^^ provided by the `TransmuteInto` trait

let _ : u32 = Foo(16, 12).transmute_into(); // Compile Error!

// error[E0277]: the trait bound `u32: TransmuteFrom<foo::Foo, _>` is not satisfied
//   --> src/demo.rs:15:27
//    |
// 15 | let _ : u32 = Foo(16, 12).transmute_into(); // Compile Error!
//    |                           ^^^^^^^^^^^^^^ the trait `TransmuteFrom<foo::Foo, _>` is not implemented for `u32`
//    |
//   = note: required because of the requirements on the impl of `TransmuteInto<u32, _>` for `foo::Foo`
```


# Motivation
[motivation]: #motivation

We foresee a range of practical consequences of fulfilling these goals, including [making unsafe Rust safer](#making-unsafe-rust-safer), providing [safer initialization primitives](#safer-initialization-primitives), and [generic atomics](#atomict). We explore these use-cases, and more, in this section.

A *comprehensive* approach to safe transmutation provides benefits beyond the mere act of transmutation; namely:
 - [authoritatively codifies language layout guarantees](#codifying-language-layout-guarantees)
 - [allows crate authors to codify their types' layout stability guarantees](#expressing-library-layout-guarantees)
 - [allows crate authors to codify their abstractions' layout requirements](#expressing-layout-requirements)

## Codifying Language Layout Guarantees
Documentation of Rust's layout guarantees for a type are often spread across countless issues, pull requests, RFCs and various official resources. It can be very difficult to get a straight answer. When transmutation is involved, users must reason about the *combined* layout properties of the source and destination types.

This RFC proposes mechanisms that programmers will use to confidently answer such questions—by checking whether the `TransmuteFrom` and `TransmuteInto` traits are implemented, or (equivalently) by checking whether the `can_transmute` predicate (a `const fn`) is satisfied.

## Expressing Library Layout Guarantees
There is no canonical way for crate authors to declare the SemVer layout guarantees of their types. Crate authors currently must state their layout guarantees using prose in their documentation. In contrast to structural stability (e.g., the declared visibility of fields), layout stability is expressed extra-linguistically.

This isn't satisfactory: guarantees expressed in prose outside of the Rust programming language are guarantees that cannot be reasoned about *inside* the language. Whereas `rustc` can dutifully deny programmers access to private fields, it is unable to prevent programmers from making unfounded expectations of types' in-memory layouts.

This RFC proposes simple-but-powerful [mechanisms][stability] for declaring layout stability guarantees.

## Expressing Layout Requirements
Similarly, there is no canonical way for crate authors to declare the layout requirements of generic abstractions over types that have certain layout properties. 

For instance, a common bit-packing technique involves abusing the relationship between allocations and alignment. If a type is aligned to 2<sup>n</sup>, then the *n* least significant bits of pointers to that type will equal `0`. These known-zero bits can be packed with data. Since alignment cannot be currently reasoned about at the type-level, it's currently impossible to bound instantiations of a generic parameter based on minimum alignment.

The mechanisms proposed by the RFC enable this. We return to this example near the end of the RFC, [here][case-study-alignment].

## Making Unsafe Rust Safer
[motivation-safer-unsafe]: #Making-Unsafe-Rust-Safer
In the blog post [*Unsafe Zig is Safer than Unsafe Rust*](https://andrewkelley.me/post/unsafe-zig-safer-than-unsafe-rust.html), Andrew Kelley asks readers to consider the following Rust code:

```rust
struct Foo {
    a: i32,
    b: i32,
}

fn main() {
    unsafe {
        let mut array: [u8; 1024] = [1; 1024];
        let foo = mem::transmute::<&mut u8, &mut Foo>(&mut array[0]);
        foo.a += 1;
    }
}
```
This pattern, Kelley notes, is common when interacting with operating system APIs.

This rust program compiles, but quietly introduces undefined behavior. `Foo` requires stricter in-memory alignment than `u8`. The equivalent code in Zig produces a compiler error for this reason and requires modification.

And that's not all: Kelley misses that the very layout of `Foo` itself is undefined⁠—it must be annotated with `#[repr(C)]`!

Kelley concludes:
> In Zig the problem of alignment is solved completely; the compiler catches all possible alignment issues. In the situation where you need to assert to the compiler that something is more aligned than Zig thinks it is, you can use `@alignCast`. This inserts a cheap safety check in debug mode to make sure the alignment assertion is correct.

This situation is frustrating. Like `zig`, `rustc` *knows* the layout of `Foo`, it *knows* that the layout of `Foo` is unspecified because it lacks the `#[repr(C)]` attribute, and it *knows* the minimum alignment requirements of `Foo` are stricter than that of `u8`. Why doesn't it also help us avoid UB?

This RFC proposes mechanisms that use this information to enable safer transmutation. We revisit Kelley's motivating example with these proposed mechanisms near the end of this RFC, [*here*][case-study-safer-unsafe].

## Efficient Network Parsing
***TODO: Josh?***

## Safer Initialization Primitives
The initialization primitives `mem::zeroed<T>` and `mem::uninitialized<T>` are `unsafe` because they may be used to initialize types for which zeroed or uninitialized memory are *not* valid bit-patterns. The `mem::zeroed` function recently gained a dynamically-enforced validity check, but this safety measure isn't wholly satisfactory: the validity properties of `T` are statically known, but the check is dynamic.

This RFC proposes mechanisms that would allow these functions to explicitly bound their generic parameter `T` based on its validity properties. We revisit this motivating use-case with our proposed mechanisms near the end of this RFC, [*here*][future-possibility-safe-initialization].

## Including Structured Data
[motivation-including-data]: #Including-Structured-Data

[`include_bytes`]: https://doc.rust-lang.org/core/macro.include_bytes.html

The [`include_bytes`] macro statically includes the contents of another file as a byte array, producing a value of type `&'static [u8; N]`. Often, these bytes correspond to structured data and a `transmute` must be used to deserialize the bytes into useful form; e.g.:
```rust
pub fn recognize(input: &Matrix<f64, U1, U784>) -> usize
{
    static RAW_WEIGHT : &'static [u8; 62_720] = include_bytes!("/weight.bin");

    static RAW_BIAS : &'static [u8; 80] = include_bytes!("/bias.bin");

    let WEIGHT: &Matrix<f64, U784, U10> = unsafe{ mem::transmute(RAW_WEIGHT) };

    let BIAS: &Matrix<f64, U1, U10> = unsafe{ mem::transmute(RAW_BIAS) };

    network::recognize(input, WEIGHT, BIAS)
}
```

This RFC proposes mechanisms that would eliminate `unsafe` in this example. We revisit this motivating use-case with our proposed mechanisms near the end of this RFC, [*here*][future-possibility-include_data].


## SIMD
***TODO:*** https://internals.rust-lang.org/t/pre-rfc-frombits-intobits/7071


## `Atomic<T>`
Rust defines a dozen `Atomic*` types (`AtomicBool`, `AtomicI8`, `AtomicI16`, `AtomicI32`, `AtomicI64`, `AtomicIsize`, `AtomicPtr`, `AtomicU8`, `AtomicU16`, `AtomicU32`, `AtomicU64`, and `AtomicUsize`).

This set is large—a distinct `Atomic*` type is required for each primitive type—but incomplete. If one wants atomic operations on their own type, they must define a wrapper around an existing `Atomic*` type of appropriate size and validity, then transmute at API boundaries; e.g.:

```rust
#[repr(u8)]
enum Trilean {
    False,
    True,
    Unknown,
}

#[repr(transparent)]
pub AtomicTrilean(AtomicU8);

impl AtomicTrilean {

    pub const fn new(v: Trilean) -> Self {
        AtomicTrilean(
          AtomicU8::new(
            unsafe { mem::transmute(v) }
          ))
    }

    ...
}
```

The mechanisms proposed by this RFC would eliminate this pattern, permitting truly generic `Atomic` types. We revisit this motivating use-case with these proposed mechanisms near the end of this RFC, [*here*][future-possibility-generic-atomics].

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Terminology & Concepts

### 📖 Transmutation
**Transmutation** is the act of reinterpreting the bytes corresponding to a value of one type as if they corresponded to a different type.

### 📖 Soundness
A transmutation is ***sound*** if the mere act of transmuting a value from one type to another is not compiler undefined behavior.

### 📖 Safety
A sound transmutation is ***safe*** if *using* the transmuted value cannot violate memory safety.

### 📖 Stability
A safe transmutation is ***stable*** if the authors of the source type and destination types have indicated that the layouts of those types is part of their libraries' stability guarantees.

## Concepts in Depth

***Disclaimer:** While the high-level definitions of transmutation soundness, safety and stability are a core component of this RFC, the detailed rules and examples in this section are **not**. We expect that the initial implementation of `TransmuteFrom` may initially be considerably less sophisticated than the examples in this section (and thus forbid valid transmutations). Nonetheless, this section explores nuanced cases of transmutation soundness and safety to demonstrate that the APIs we propose can grow to handle that nuance.*


### 📖 When is a transmutation sound?
[sound transmutation]: #When-is-a-transmutation-sound?
A transmutation is ***sound*** if the mere act of transmuting a value from one type to another is not undefined behavior.

#### Well-Defined Representation
[`u8`]: core::u8
[`f32`]: core::f32

Transmutation is *always unsound* if it occurs between types with unspecified representations.

Most of Rust's primitive types have specified representations. That is, the precise layout characteristics of [`u8`], [`f32`] is a documented and guaranteed aspect of those types.

In contrast, most `struct` and `enum` types defined without an explicit `#[repr(C)]` attribute do ***not*** have well-specified layout characteristics.

To ensure that types you've define are soundly transmutable, you almost always (with very few exceptions) must mark them with the `#[repr(C)]` attribute.

#### Requirements on Owned Values
[transmute-owned]: #transmuting-owned-values

Transmutations involving owned values must adhere to two rules to be sound. They must:
 * [preserve or broaden the bit validity][owned-validity], and
 * [preserve or shrink the size][owned-size].

##### Preserve or Broaden Bit Validity
[owned-validity]: #Preserve-or-Broaden-Bit-Validity
[`NonZeroU8`]: https://doc.rust-lang.org/beta/core/num/struct.NonZeroU8.html

The bits of any valid instance of the source type must be a bit-valid instance of the destination type.

For example, we are permitted to transmute a `Bool` into a [`u8`]:
```rust
#[derive(Default, PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(u8)]
enum Bool {
    True = 1,
    False = 0,
}

let _ : u8 = Bool::True.transmute_into();
let _ : u8 = Bool::False.transmute_into();
```
<sup>
(Note: <code>#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]</code> annotation connotes that <i>all</i> aspects of <code>Bool</code>'s layout are part of its <a href="#When-is-a-transmutation-stable?">library stability guarantee</a>.)
</sup>

...because all possible instances of `Bool` are also valid instances of [`u8`]. However, transmuting a [`u8`] into a `Bool` is forbidden:
```rust
/* ⚠️ This example intentionally does not compile. */
let _ : Bool = u8::default().transmute_into(); // Compile Error!
```
...because not all instances of [`u8`] are valid instances of `Bool`.

Another example: While laying out certain types, Rust may insert padding bytes between the layouts of fields. In the below example `Padded` has two padding bytes, while `Packed` has none:
```rust
#[repr(C)]
#[derive(Default, PromiseTransmutableFrom, PromiseTransmutableInto)]
struct Padded(pub u8, pub u16, pub u8);

#[repr(C)]
#[derive(Default, PromiseTransmutableFrom, PromiseTransmutableInto)]
struct Packed(pub u16, pub u16, pub u16);

assert_eq!(mem::size_of::<Packed>(), mem::size_of::<Padded>());
```

We may safely transmute from `Packed` to `Padded`:
```rust
let _ : Padded = Packed::default().transmute_into();
```
...but not from `Padded` to `Packed`:
```rust
/* ⚠️ This example intentionally does not compile. */
let _ : Packed = Padded::default().transmute_into(); // Compile Error!
```
...because doing so would expose two uninitialized padding bytes in `Padded` as if they were initialized bytes in `Packed`.

##### Preserve or Shrink Size
[owned-size]: #Preserve-or-Shrink-Size

It's completely sound to transmute into a type with fewer bytes than the source type; e.g.:
```rust
let _ : [u8; 16] = [u8; 32]::default().transmute_into();
```
This transmute truncates away the final sixteen bytes of the `[u8; 32]` value.

A value may ***not*** be transmuted into a type of greater size, if doing so would expose uninitialized bytes as initialized:
```rust
/* ⚠️ This example intentionally does not compile. */
let _ : [u8; 32] = [u8; 16]::default().transmute_into(); // Compile Error!
```

#### Requirements on References
[transmute-references]: #transmuting-references

The [restrictions above that apply to transmuting owned values][transmute-owned] also apply to transmuting references. However, references carry a few additional restrictions.

A [sound transmutation] must:
 - [preserve or shrink size][reference-size],
 - [preserve or relax alignment][reference-alignment],
 - [preserve or shrink lifetimes][reference-lifetimes],
 - [preserve or shrink uniqueness][reference-mutability], and
 - and if the destination type is a mutate-able reference, [preserve validity][reference-validity].

##### Preserve or Shrink Size
[reference-size]: #Preserve-or-Shrink-Size

You may preserve or decrease the size of the referent type via transmutation:
```rust
let _: &[u8; 3] = (&[0u8; 9]).transmute_into();
```

However, you may **not**, under any circumstances, *increase* the size of the referent type:
```rust
/* ⚠️ This example intentionally does not compile. */
let _: &[u8; 9] = (&[0u8; 3]).transmute_into(); // Compile Error! 
```
##### Preserve or Relax Alignment
[reference-alignment]: #Preserve-or-Relax-Alignment

Unaligned loads are undefined behavior. You may transmute a reference into reference of more relaxed alignment:
```rust
let _: &[u8; 0] = (&[0u16; 0]).transmute_into();
```

However, you may **not** transmute a reference into a reference of more-restrictive alignment:
```rust
/* ⚠️ This example intentionally does not compile. */
let _: &[u16; 0] = (&[0u8; 0]).transmute_into(); // Compile Error! 
```

##### Preserve or Shrink Lifetimes
[reference-lifetimes]: #Preserve-or-Shrink-Lifetimes

You may transmute a reference into a reference of lesser lifetime:
```rust
fn shrink<'a>() -> &'a u8 {
    static long : &'static u8 = &16;
    long.transmute_into()
}
```

However, you may **not** transmute a reference into a reference of greater lifetime:
```rust
/* ⚠️ This example intentionally does not compile. */
fn extend<'a>(short: &'a u8) -> &'static u8 {
    short.transmute_into() // Compile Error!
}
```

##### Preserve or Shrink Uniqueness
[reference-mutability]: #Preserve-or-Shrink-Uniqueness

You may preserve or decrease the uniqueness of a reference through transmutation:
```rust
let _: &u8 = (&42u8).transmute_into();
let _: &u8 = (&mut 42u8).transmute_into();
```

However, you may **not** transmute a shared reference into a unique reference:
```rust
/* ⚠️ This example intentionally does not compile. */
let _: &mut u8 = (&42u8).transmute_into(); // Compile Error!
```

##### Mutate-able References Must Preserve Validity
[reference-validity]: #Mutate-able-References-Must-Preserve-Validity

A mutate-able reference is:
- all unique (i.e., `&mut T`) references
- all shared (i.e., `&T`) references whose referent type contain any bytes produced by the contents of `UnsafeCell`.

Unlike transmutations of owned values, the transmutation of a mutate-able reference may also not expand the bit-validity of the referenced type. For instance:
```rust
/* ⚠️ This example intentionally does not compile. */
let mut x = NonZeroU8::new(42).unwrap();
{
    let y : &mut u8 = (&mut x).transmute_into(); // Compile Error!
    *y = 0;
}

let z : NonZeroU8 = x;
```
If this example did not produce a compile error, the value of `z` would not be a bit-valid instance of its type, [`NonZeroU8`].



### 📖 When is a transmutation safe?
A sound transmutation is ***safe*** if *using* the transmuted value safely cannot violate memory safety. Whereas soundness solely concerns the act of transmutation, *safety* is concerned with what might happen with a value *after* transmutation occurs.

#### Implicit Constructability
A struct or enum variant is *fully implicitly constructable* at a given location only if, at that location, that type can be instantiated via its *implicit constructor*, and its fields are also *implicitly constructable*.

The *implicit constructor* of a struct or enum variant is the constructor Rust creates implicitly from its definition; e.g.:
```rust
struct Point<T> {
    x: T,
    y: T,
}

let p = Point { x: 4, y: 2 };
     // ^^^^^^^^^^^^^^^^^^^^ An instance of `Point` is created here, via its implicit constructor.
```

Limiting implicit constructability is the fundamental mechanism with which type authors build safe abstractions for `unsafe` code, whose soundness is dependent on preserving invariants on fields. Usually, this takes the form of restricting the visibility of fields. For instance, consider the type `NonEmptySlice`, which enforces a validity constraint on its fields via its constructor:

```rust
pub mod crate_a {

    #[repr(C)]
    pub struct NonEmptySlice<'a, T> {
        data: *const T,
        len: usize,
        lifetime: core::marker::PhantomData<&'a ()>,
    }

    impl<'a, T> NonEmptySlice<'a, T> {
        pub fn from_array<const N: usize>(arr: &'a [T; N], len: usize) -> Self {
            assert!(len <= N);
            assert!(len > 0);
            Self {
                data: arr as *const T,
                len,
                lifetime: core::marker::PhantomData,
            }
        }

        pub fn first(&self) -> &'a T {
            unsafe { &*self.data }
        }
    }

}
```
It is sound for `first` to be a *safe* method is because the `from_array` constructor ensures that `data` is safe to dereference, and because `from_array` is the *only* way to safely initialize `NonEmptySlice` outside of `crate_a` (note that `NonEmptySlice`'s fields are *not* `pub`). As a rule: any field that is not marked `pub` should be assumed to be private *because* it is subject to safety invariants.

Unfortunately, field visibility modifiers are not a surefire indicator of whether a type is *fully* implicitly constructable. A type author may restrict the implicit constructability of a type even in situations where all fields of that type (*and all fields of those fields*) are `pub`; consider:
```rust
pub mod crate_a {

    #[repr(C)]
    pub struct NonEmptySlice<'a, T>(pub private::NonEmptySliceInner<'a, T>);

    impl<'a, T> NonEmptySlice<'a, T> {
        pub fn from_array<const N: usize>(arr: &'a [T; N], len: usize) -> Self {
            assert!(len <= N && len > 0);
            Self(
                private::NonEmptySliceInner {
                    data: arr as *const T,
                    len,
                    lifetime: core::marker::PhantomData,
                }
            )
        }

        pub fn first(&self) -> &'a T {
            unsafe { &*self.0.data }
        }
    }

    // introduce a private module to avoid `private_in_public` error (E0446):
    pub(crate) mod private {
        #[repr(C)]
        pub struct NonEmptySliceInner<'a, T> {
            pub data: *const T,
            pub len: usize,
            pub lifetime: core::marker::PhantomData<&'a ()>,
        }
    }

}
```
In the above example, the definitions of both `NonEmptySlice` and its field `NonEmptySliceInner` are marked `pub`, and all fields of these types are marked `pub`. However, `NonEmptySlice` is *not* fully implicitly constructible outside of `crate_a`, because the module containing `NonEmptySliceInner` is not visibile outside of `crate_a`.

#### Constructability and Transmutation
Transmutation supplies a mechanism for constructing instances of a type *without* invoking its implicit constructor, nor any constructors defined by the type's author.

In the previous examples, it would be *unsafe* to transmute `0u128` into `NonEmptySlice` outside `crate_a`, because subsequent *safe* use of that value (namely, calling `first`) would violate memory safety. (However, it's completely safe to transmute `NonEmptySlice` into a `u128`.)

For transmutations where the destination type involves mutate-able references, the constructability of the source type is also relevant. Consider:
```rust
/* ⚠️ This example intentionally does not compile. */
let arr = [0u8, 1u8, 2u8];
let mut x = NonEmptySlice::from_array(&arr, 2);
{
    let y : &mut u128 = (&mut x).transmute_into(); // Compile Error!
    *y = 0u128;
}

let z : NonEmptySlice<u8> = x;
```
If this example did not produce a compile error, the value of `z` would not be a safe instance of its type, `NonEmptySlice`, because `z.first()` would dereference a null pointer.

### 📖 When is a transmutation stable?
[stability]: #When-is-a-transmutation-stable

Since the soundness and safety of a transmutation is affected by the layouts of the source and destination types, changes to those types' layouts may cause code which previously compiled to produce errors. In other words, transmutation causes a type's layout to become part of that type's API for the purposes of SemVer stability.

The question is, then: *how can the author of a type reason about transmutations they did not write, from-or-to types they did not write?* We address this problem by introducing two traits which both allow an author to opt-in to stability guarantees for their types, and allow third-parties to reason at compile-time about what guarantees are provided for such types.

#### `PromiseTransmutableFrom` and `PromiseTransmutableInto`

You may declare the stability guarantees of your type by implementing one or both of two traits:
```rust
pub trait PromiseTransmutableFrom
{
    type Archetype
        : TransmuteInto<Self, NeglectStability>
        + PromiseTransmutableFrom;
}

pub trait PromiseTransmutableInto
{
    type Archetype
        : TransmuteFrom<Self, NeglectStability>
        + PromiseTransmutableInto;
}
```

To implement each of these traits, you must specify an `Archetype`. An `Archetype` is a type whose layout exemplifies the extremities of your stability promise (i.e., the least/most constrained type for which it is valid to transmute your type into/from).

By implementing `PromiseTransmutableFrom`, you promise that your type is guaranteed to be safely transmutable *from* `PromiseTransmutableFrom::Archetype`. Conversely, by implementing `PromiseTransmutableInto`, you promise that your type is guaranteed to be safely transmutable *into* `PromiseTransmutableInto::Archetype`.

You are free to change the layout of your type however you like between minor crate versions so long as that change does not violates these promises. These two traits are capable of expressing simple and complex stability guarantees.

#### Stability & Transmutation
A `Src` type is *stably* transmutable into a `Dst` type *only if* `<Src as PromiseTransmutableInto>::Archetype` is transmutable, stability notwithstanding, into `<Dst as PromiseTransmutableFrom>::Archetype`; formally:
```rust
unsafe impl<Src, Dst> TransmuteFrom<Src> for Dst
where
    Src: PromiseTransmutableInto,
    Dst: PromiseTransmutableFrom,
    <Dst as PromiseTransmutableFrom>::Archetype:
        TransmuteFrom<
            <Src as PromiseTransmutableInto>::Archetype,
            NeglectStability
        >
{}
```

Why is this the case?

The type `<Src as PromiseTransmutableInto>::Archetype` exemplifies the furthest extreme of non-breaking changes that could be made to the layout of `Src` that could affect its use as a source type in transmutations. Conversely, `<Dst as PromiseTransmutableFrom>::Archetype` exemplifies the furthest extreme of non-breaking changes that could be made to the layout of `Dst` that could affect its use as a destination type in transmutations. If a transmutation between these extremities is valid, then so is `Src: TransmuteInto<Dst>`.

#### Common Use-Case: As-Stable-As-Possible
[stability-common]: #common-use-case-as-stable-as-possible
To promise that all transmutations which are currently safe for your type will remain so in the future, simply annotate your type with:
```rust
#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(C)]
pub struct Foo(pub Bar, pub Baz);
```
This expands to:
```rust
#[repr(C)]
pub struct Foo(pub Bar, pub Baz);

/// Generated `PromiseTransmutableFrom` for `Foo`
const _: () = {
    use core::transmute::stability::PromiseTransmutableFrom;

    #[repr(C)]
    pub struct TransmutableFromArchetype(
        pub <Bar as PromiseTransmutableFrom>::Archetype,
        pub <Baz as PromiseTransmutableFrom>::Archetype,
    );

    impl PromiseTransmutableFrom for Foo {
        type Archetype = TransmutableFromArchetype;
    }
};

/// Generated `PromiseTransmutableInto` for `Foo`
const _: () = {
    use core::transmute::stability::PromiseTransmutableInto;

    #[repr(C)]
    pub struct TransmutableIntoArchetype(
        pub <Bar as PromiseTransmutableInto>::Archetype,
        pub <Baz as PromiseTransmutableInto>::Archetype,
    );

    impl PromiseTransmutableInto for Foo {
        type Archetype = TransmutableIntoArchetype;
    }
};
```
Since deriving *both* of these traits together is, by far, the most common use-case, we [propose][extension-promisetransmutable-shorthand] `#[derive(PromiseTransmutable)]` as an ergonomic shortcut.

#### Uncommon Use-Case: Weak Stability Guarantees
[stability-uncommon]: #uncommon-use-case-weak-stability-guarantees

If you are *most people* and want to declare your types as layout-stable, you should follow the advice in the previous sections. In doing so, you declare that you will *not* modify the layout of your type in virtually any way, except as a breaking change. If your type's fields have simple stability guarantees, this effects the strongest possible declaration of stability: it declares that *all* transmutations that are safe are *also* stable.

However, if you anticipate that you will need to make changes to your type's layouts in a non-breaking way, you can craft uninformative archetypes that effect *weaker* promises of stability. With uninformative archetypes, only a *subset* of transmutations that are safe are also considered stable. For instance, you may use `PromiseTransmutableInto` to specify an lower-bound for the size of a type, but *not* make promises about its bit-validity:

```rust
#[repr(C)] pub struct Foo(u8, u8, u8, u8, u8, u8);

#[repr(C)]
struct FooArchetype(pub [MaybeUninit<u8>; 5]);

impl PromiseTransmutableInto for Foo {
    type Type = [MaybeUninit<u8>; 5];
}
```

##### Crafting Uninformative `PromiseTransmutableFrom` Archetypes

###### Archetype Alignment
You may freely decrease the required alignment of your type to any value less-than-or-equal to the required alignment of your archetype. The maximum alignment requirement permitted by Rust is 2<sup>29</sup>.

###### Archetype Size
You may decrease the size of your type without violating stability to any size less-than-or-equal to the size of your archetype. Unfortunately, the maximum size of types in Rust is [architecture-dependent](https://github.com/rust-lang/rust/blob/63b441aafbf52d6ba789ecc478455800c1a48df9/src/librustc_target/abi/mod.rs#L179-L197). For the greatest portability, your archetype should be no larger than 2<sup>15</sup>. An archetype of this size provides (un)reasonable future flexibility.

###### Archetype Validity & Visibility
You may freely make your type *less* constructible (in terms of both theoretical validity and constructor visibility) than your archetype.

A minimally-informative archetype may be constructed using this as a building-block:
```rust
struct None(u8);
```
Since the `u8` field is private, its valid values are assumed to be constrained by library invariants.

##### Crafting Uninformative `PromiseTransmutableInto` Archetypes

###### Archetype Alignment
You may freely increase the required alignment of your type to any value greater-than-or-equal to the required alignment of your archetype. The minimum alignment requirement permitted by Rust is 1.

###### Archetype Size
You may decrease the size of your type without violating stability to any size less-than-or-equal to the size of your archetype. The minimum size of types in Rust is 0.

###### Archetype Validity & Visibility
You may freely make your type more constructible (in terms of both visibility and theoretical validity) than your archetype. A minimally-informative archetype may be constructed using `MaybeUninit<u8>` as a building-block, since *any* possible instantiation of a byte is a valid instantiation of `MaybeUninit<u8>`.


## Mechanisms of Transmutation

Two traits provide mechanisms for transmutation between types: 
```rust
// this trait is implemented automagically by the compiler
#[lang = "transmute_from"]
pub unsafe trait TransmuteFrom<Src: ?Sized, Neglect = ()>
where
    Neglect: UnsafeTransmuteOptions,
{
    #[inline(always)]
    fn transmute_from(src: Src) -> Self
    where
        Src: Sized,
        Self: Sized,
        Neglect: SafeTransmuteOptions,
    {
        unsafe { Self::unsafe_transmute_from(src) }
    }

    #[inline(always)]
    unsafe fn unsafe_transmute_from(src: Src) -> Self
    where
        Src: Sized,
        Self: Sized,
        Neglect: UnsafeTransmuteOptions,
    {
        use core::{mem, ptr};
        unsafe {
            let dst = ptr::read_unaligned(&src as *const Src as *const Self);
            mem::forget(src);
            dst
        }
    }
}

// implemented in terms of `TransmuteFrom`
pub unsafe trait TransmuteInto<Dst: ?Sized, Neglect = ()>
where
    Neglect: UnsafeTransmuteOptions,
{
    fn transmute_into(self) -> Dst
    where
        Self: Sized,
        Dst: Sized,
        Neglect: SafeTransmuteOptions;

    unsafe fn unsafe_transmute_into(self) -> Dst
    where
        Self: Sized,
        Dst: Sized,
        Neglect: UnsafeTransmuteOptions;
}

unsafe impl<Src, Dst, Neglect> TransmuteInto<Dst, Neglect> for Src
where
    Src: ?Sized,
    Dst: ?Sized + TransmuteFrom<Src, Neglect>,
    Neglect: UnsafeTransmuteOptions,
{
    ...
}
```

In the above definitions, `Src` represents the source type of the transmutation, `Dst` represents the destination type of the transmutation, and `Neglect` is a parameter that [encodes][options] which static checks the compiler ought to neglect when considering if a transmutation is valid. The default value of `Neglect` is `()`, which reflects that, by default, the compiler does not neglect *any* static checks.

### Neglecting Static Checks
[options]: #Neglecting-Static-Checks

The default value of the `Neglect` parameter, `()`, statically forbids transmutes that are unsafe, unsound, or unstable. However, you may explicitly opt-out of some static checks:

| Transmute Option    | Compromises | Usable With                                             |
|---------------------|-------------|---------------------------------------------------------|
| `NeglectStabilty`   | Stability   | `transmute_{from,into}`, `unsafe_transmute_{from,into}` |
| `NeglectAlignment`  | Safety      | `unsafe_transmute_{from,into}`                          |
| `NeglectValidity`   | Soundness   | `unsafe_transmute_{from,into}`                          |

`NeglectStabilty` implements the `SafeTransmuteOptions` and `UnsafeTransmuteOptions` marker traits, as it can be used in both safe and unsafe code. The selection of multiple options is encoded by grouping them as a tuple; e.g., `(NeglectAlignment, NeglectValidity)` is a selection of both the `NeglectAlignment` and `NeglectValidity` options.

We introduce two marker traits which serve to group together the options that may be used with safe transmutes, and those which may be used with `unsafe` transmutes:
```rust
pub trait SafeTransmuteOptions: private::Sealed
{}

pub trait UnsafeTransmuteOptions: SafeTransmuteOptions
{}

impl SafeTransmuteOptions for () {}
impl UnsafeTransmuteOptions for () {}
```

#### `NeglectStability`
[`NeglectStability`]: #neglectstability

By default, `TransmuteFrom` and `TransmuteInto`'s methods require that the [layouts of the source and destination types are SemVer-stable][stability]. The `NeglectStability` option disables this requirement.
```rust
pub struct NeglectStability;

impl SafeTransmuteOptions for NeglectStability {}
impl UnsafeTransmuteOptions for NeglectStability {}
```

Prior to the adoption of the [stability declaration traits][stability], crate authors documented the layout guarantees of their types with doc comments. The `TransmuteFrom` and `TransmuteInto` traits and methods may be used with these types by requesting that the stability check is neglected; for instance:

```rust
fn serialize<W: Write>(val : LibraryType, dst: W) -> std::io::Result<()>
where
    LibraryType: TransmuteInto<[u8; size_of::<LibraryType>()], NeglectStability>
{
    ...
}
```

Neglecting stability over-eagerly cannot cause unsoundness or unsafety. For this reason, it is the only transmutation option available on the safe methods `transmute_from` and `transmute_into`. However, neglecting stability over-eagerly may cause your code to cease compiling if the authors of the source and destination types make changes that affect their layout.

By using the `NeglectStability` option to transmute types you do not own, you are committing to ensure that your reliance on these types' layouts is consistent with their documented stability guarantees.

#### `NeglectAlignment`
By default, `TransmuteFrom` and `TransmuteInto`'s methods require that, when transmuting references, the minimum alignment of the destination's referent type is no greater than the minimum alignment of the source's referent type. The `NeglectAlignment` option disables this requirement.
```rust
pub struct NeglectAlignment;

impl UnsafeTransmuteOptions for NeglectAlignment {}
```

By using the `NeglectAlignment` option, you are committing to ensure that the transmuted reference satisfies the alignment requirements of the destination's referent type. For instance:
```rust
/// Try to convert a `&T` into `&U`.
///
/// This produces `None` if the referent isn't appropriately
/// aligned, as required by the destination type.
pub fn try_cast_ref<'t, 'u, T, U>(src: &'t T) -> Option<&'u U>
where
    &'t T: TransmuteInto<&'u U, NeglectAlignment>,
{
    if (src as *const T as usize) % align_of::<U>() != 0 {
        None
    } else {
        // Safe because we dynamically enforce the alignment
        // requirement, whose static check we chose to neglect.
        Some(unsafe { src.unsafe_transmute_into() })
    }
}
```

#### `NeglectValidity`
By default, `TransmuteFrom` and `TransmuteInto`'s methods require that all instantiations of the source type are guaranteed to be valid instantiations of the destination type. This precludes transmutations which *might* be valid depending on the source value:
```rust
#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(u8)]
enum Bool {
    True = 1,
    False = 0,
}

/* ⚠️ This example intentionally does not compile. */
let _ : Bool  = some_u8_value.transmute_into(); // Compile Error!
```
The `NeglectValidity` option disables this check.
```rust
pub struct NeglectValidity;

impl UnsafeTransmuteOptions for NeglectValidity {}
```

By using the `NeglectValidity` option, you are committing to ensure dynamically source value is a valid instance of the destination type. For instance:
```rust
#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(u8)]
enum Bool {
    True = 1,
    False = 0,
}

pub trait TryIntoBool
{
    fn try_into_bool(self) -> Option<Bool>;
}

impl<T> TryIntoBool for T
where
    T: TransmuteInto<u8>,
    u8: TransmuteInto<Bool, NeglectValidity>
{
    fn try_into_bool(self) -> Option<Bool> {
        let val: u8 = self.transmute_into();

        if val > 1 {
            None
        } else {
            // Safe, because we've first verified that
            // `val` is a bit-valid instance of a boolean.
            Some(unsafe {val.unsafe_transmute_into()})
        }
    }
}
```

Even with `NeglectValidity`, the compiler will statically reject transmutations that cannot possibly be valid:
```rust
#[derive(PromiseTransmutableInto)]
#[repr(C)] enum Foo { A = 24 }

#[derive(PromiseTransmutableFrom)]
#[repr(C)] enum Bar { Z = 42 }

let _ = <Bar as TransmuteFrom<Foo, NeglectValidity>::unsafe_transmute_from(Foo::N) // Compile error!
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


## Implementation Guidance
The *only* item defined by this RFC requiring special compiler support is `TransmuteFrom`. To realize this RFC's proposal of safe transmutation between different types, this item will require compiler support. However, the *most* minimal acceptable implementation of `TransmuteFrom` can be achieved entirely in-language:

```rust
/// A type is transmutable into itself.
unsafe impl<T, Neglect> TransmuteFrom<T, Neglect> for T
where
    Neglect: UnsafeTransmuteOptions
{}

/// A transmutation is *stable* if...
unsafe impl<Src, Dst> TransmuteFrom<Src> for Dst
where
    Src: PromiseTransmutableInto,
    Dst: PromiseTransmutableFrom,
    <Dst as PromiseTransmutableFrom>::Archetype:
        TransmuteFrom<
            <Src as PromiseTransmutableInto>::Archetype,
            NeglectStability
        >
{}
```
This [minimal implementation][minimal-impl] is sufficient for convincing the compiler to accept basic stability declarations, such as those of Rust's primitive types. It is *insufficient* for making the compiler accept transmutations between *different* types (and, consequently, complex stability declarations). Implementers should use this as a starting point.

### Listing for Initial, Minimal Implementation
[minimal-impl]: #Listing-for-Initial-Minimal-Implementation
**[This module listing provides the minimal-viable implementation of this RFC (excepting the automatic derives).](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=25a83d8d8e33e9f631a64b53e3c39844)** This listing is also the **canonical specification** of this RFC's public API surface.

### Towards an Initial, Smart Implementation

To support transmutations between different types, implementers of this RFC should begin by defining a `transmute_from` lang item to annotate libcore's definition of `TransmuteFrom`. Whether `TransmuteFrom` is implemented for a given type and parameters shall be determined within the implementation of the type system (à la [`Sized`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyS.html#method.is_sized) and [`Freeze`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyS.html#method.is_freeze)).

This initial smart implementation may be made simpler by:
  - *not* supporting enums
  - *not* supporting unions
  - *not* supporting `Neglect` parameters besides `()` and `NeglectStability`
  - simplifying constructability

#### Simplifying Constructability
The safety property of constructability defined in the guidance-level examplation of this RFC describes a platonic ideal of the property. 

However, we recognize that this definition poses implementation challenges: In our definition of constructability, answering the question of whether a struct or enum variant is constructible depends on *where* that question is being asked. Consequently, answering whether a given type `Src` is `TransmutableInto` a given type `Dst` will depend on *where* that question is posed.

We recommend adopting a simplified definition of *constructability*: a type is *constructible* if its fields are marked `pub`, and those fields are constructible. With this definition, answering the question of whether a type is constructible does *not* depend on where the question is asked: we do not examine the visibility of the involved types.

Unfortunately, with no other actions taken, this simplified definition comes...

##### ...at the Cost of Safety
This definition is *usually* sufficient for ensuring safety: it is *generally* an error to expose a private type in a public type signature. However, these errors may be circumvented using the public-type-in-private-module trick:
```rust
pub mod crate_a {

    #[repr(C)]
    pub struct NonEmptySlice<'a, T>(pub private::NonEmptySliceInner<'a, T>);

    impl<'a, T> NonEmptySlice<'a, T> {
        pub fn from_array<const N: usize>(arr: &'a [T; N], len: usize) -> Self {
            assert!(len <= N && len > 0);
            Self(
                private::NonEmptySliceInner {
                    data: arr as *const T,
                    len,
                    lifetime: core::marker::PhantomData,
                }
            )
        }

        pub fn first(&self) -> &'a T {
            unsafe { &*self.0.data }
        }
    }

    // introduce a private module to avoid `private_in_public` error (E0446):
    pub(crate) mod private {
        #[repr(C)]
        pub struct NonEmptySliceInner<'a, T> {
            pub data: *const T,
            pub len: usize,
            pub lifetime: core::marker::PhantomData<&'a ()>,
        }
    }

}
```
With this simplified definition of constructability, it is possible for a third-party to define a *safe* constructor of `NonEmptySlice` that produces a value which is *unsafe* to use:
```rust
pub evil_constructor<T>(src: T) -> NonEmptySlice<'static, u8>
where
    T: TransmuteInto<NonEmptySlice<'static, u8>, NeglectStability>,
{
    src.transmute_into()
}

evil_constructor(0u128).first() // muahaha!
```

The above code is "safe" because our simplified definition of constructability fails to recognize this pattern of encapsulation, and because `NeglectStability` is a `SafeTransmutationOption`.

The intent of `NeglectStability` is to permit the safe transmutation of types that predate the stabilization of the stability declaration traits. It also provides a convenient escape-hatch for type authors to neglect the stability of transmutations of their *own* types, without sacrificing safety. `NeglectStability` is a `SafeTransmutationOption` because, in principle, neglecting stability does not diminish safety. Our simplified definition of constructability violates this principle.

By temporarily sacrificing these goals, we may preserve safety solely...

##### ...at the Cost of `NeglectStability`
We may preserve safety by demoting `NeglectStability` to `UnsafeTransmutationOption`-status.

In doing so, a third-party is forced to resort to an `unsafe` transmutation to construct `NonEmptySlice`; e.g.:

```rust
pub evil_constructor<T>(src: T) -> NonEmptySlice<'static, u8>
where
    T: TransmuteInto<NonEmptySlice<'static, u8>, NeglectStability>,
{
    // unsafe because we `NeglectStability`
    unsafe { src.unsafe_transmute_into() }
}
```

Demoting `NeglectStability` to unsafe-status does not stop type authors from opting-in to stable (and thus safe) transmutations; e.g., with `derive(PromiseTransmutableFrom)`:
```rust
pub mod crate_a {

    #[derive(PromiseTransmutableFrom)]
    #[repr(C)]
    pub struct NonEmptySlice<'a, T>(pub private::NonEmptySliceInner<'a, T>);

    impl<'a, T> NonEmptySlice<'a, T> {
        pub fn from_array<const N: usize>(arr: &'a [T; N], len: usize) -> Self {
            assert!(len <= N && len > 0);
            Self(
                private::NonEmptySliceInner {
                    data: arr as *const T,
                    len,
                    lifetime: core::marker::PhantomData,
                }
            )
        }

        pub fn first(&self) -> &'a T {
            unsafe { &*self.0.data }
        }
    }

    // introduce a private module to avoid `private_in_public` error (E0446):
    pub(crate) mod private {
        #[derive(PromiseTransmutableFrom)]
        #[repr(C)]
        pub struct NonEmptySliceInner<'a, T> {
            pub data: *const T,
            pub len: usize,
            pub lifetime: core::marker::PhantomData<&'a ()>,
        }
    }

}
```
In the above example, the type author declares `NonEmptySlice` and `NonEmptySliceInner` to be stably instantiatable via transmutation. Given this, a third-party no longer needs to resort to `unsafe` code to violate the the invariants on `inner`:
```rust
pub evil_constructor<T>(src: T) -> NonEmptySlice<'static, u8>
where
    T: TransmuteInto<NonEmptySlice<'static, u8>>,
{
    src.transmute_into()
}

evil_constructor(0u128).first() // muahaha!
```
This safety hazard is not materially different from the one that would be induced if the type author implemented `DerefMut<Target=NonEmptySliceInner>` for `NonEmptySlice`, or made the `private` module `pub`, or otherwise explicitly provided outsiders with unrestricted mutable access to `data`.

##### Recommendation
We recommend that that implementers of this RFC initially simplify constructability by:
 - adopting our simplified definition of constructability
 - demoting `NeglectStability` to unsafe status (i.e., not implementing `SafeTransmuteOptions` for `NeglectStability`; *only* `UnsafeTransmuteOptions`)

If and when the implementation of `TransmuteFrom` encodes our complete definition of constructability, `NeglectStability` shall become a safe transmute option.


## Dissecting Stability

In this section, we iteratively review the design of the stability declaration traits, starting with a simplest-possible design:
```rust
pub unsafe trait PromiseTransmutableFrom
{
    type Archetype;
}

pub unsafe trait PromiseTransmutableInto
{
    type Archetype;
}
```

### Part 1: Towards Safety

#### Forbidding Archetypes with Nonsense Validity
The simplest definitions of the stability declaration traits *must* be `unsafe` because `Archetype` can be instantiated in ways that undermine soundness; e.g.:
```rust
unsafe impl PromiseTransmutableInto for u8 {
    type Archetype = u16; // <- Nonsense!
}

unsafe impl PromiseTransmutableFrom for u16 {
    type Archetype = u16;
}
```
These declarations subvert safety! Given these declarations, Rust will accept `u8: TransmuteInto<u16>`. This is problematic: `u8` is smaller than `u16`.

---

<details style="background-color:#d9edf7;padding:1em">
<summary>Why would this transmutation be accepted? (Click to Expand)</summary>

Let's solve this trait bound step-by-step:
```rust
/// 0. Initial equation:
u8:
    TransmuteInto<&'a Aligned8>
```

First, recall that a `Src` type is *stably* transmutable into a `Dst` type *only if* `<Src as PromiseTransmutableInto>::Archetype` is transmutable into `<Dst as PromiseTransmutableFrom>::Archetype`, notwithstanding stability. The layouts of `Src` and `Dst` themselves are, in theory, immaterial — all that matters are the layouts of their stability archetypes. Formally:
```rust
𝙞𝙛
    <Src as PromiseTransmutableInto>::Archetype:
        TransmuteInto<
            <Dst as PromiseTransmutableFrom>::Archetype,
            NeglectStability
        >
𝙩𝙝𝙚𝙣
    Src: TransmuteInto<Dst>
```

We apply this rule to our initial equation:
```rust
𝙞𝙛
    <u8 as PromiseTransmutableInto>::Archetype:
        TransmuteInto<
            <u16 as PromiseTransmutableFrom>::Archetype,
            NeglectStability
        >
𝙩𝙝𝙚𝙣
    u8: TransmuteInto<u16>
```

Next, recall the definitions of `PromiseTransmutableInto` for `u8` and `PromiseTransmutableFrom` for `u16`:
```rust
unsafe impl PromiseTransmutableInto for u8 {
    type Archetype = u16; // <- Nonsense!
}

unsafe impl PromiseTransmutableFrom for u16 {
    type Archetype = u16;
}
```
We apply these definitions to evaluate `<u8 as PromiseTransmutableInto>::Archetype` and `<u16 as PromiseTransmutableFrom>::Archetype`:
```rust
𝙞𝙛
    u16:
        TransmuteInto<
            u16,
            NeglectStability
        >
𝙩𝙝𝙚𝙣
    u8: TransmuteInto<u16>
```
Since `u16` can be safely transmuted into `u16`, the compiler accepts the transmutation of `u8` into `u16`!

</details>

---

We can prevent  nonsense archetypes by demanding that `Archetype` is transmutable into `Self` (notwithstanding stability):
```rust
pub unsafe trait PromiseTransmutableFrom
where
    Self::Archetype: TransmuteInto<Self, NeglectStability>,
{
    type Archetype;
}

pub unsafe trait PromiseTransmutableInto
where
    Self::Archetype: TransmuteFrom<Self, NeglectStability>,
{
    type Archetype;
}
```

Our previous attempt to skirt soundness now produces a compile error:
```rust
unsafe impl PromiseTransmutableInto for u8 {
    type Archetype = u16;
                  // ^^^ the trait `TransmuteInto<u8, _>` is not implemented for `u16`
}
```

#### Forbidding Archetypes with Nonsense Alignment
Unfortunately, the above definition is still flawed. Consider these stability declarations:

```rust
#[derive(PromiseTransmutableFrom)]
#[repr(C)]
pub struct Aligned1;

#[derive(PromiseTransmutableInto)]
#[repr(C, align(8))]
pub struct Aligned8;

unsafe impl PromiseTransmutableInto for Aligned1 {
    type Archetype = Aligned8;
}

unsafe impl PromiseTransmutableFrom for Aligned8 {
    type Archetype = Aligned1;
}
```

These declarations subvert safety! Given these declarations, Rust would accept the bound `&Aligned1: TransmuteInto<&Aligned8>`. This is problematic: `Aligned8` has *stricter* alignment requirements than `Aligned1`.

---

<details style="background-color:#d9edf7;padding:1em">
<summary>Why would this transmutation be accepted? (Click to Expand)</summary>

Let's solve this trait bound step-by-step:
```rust
/// 0. Initial equation:
for<'a> &'a Aligned1:
        TransmuteInto<&'a Aligned8>
```

As before, we first apply the stability rule to our equation:
```rust
𝙞𝙛
    <&'a Aligned1 as PromiseTransmutableInto>::Archetype:
        TransmuteInto<
            <&'a Aligned8 as PromiseTransmutableFrom>::Archetype,
            NeglectStability
        >
𝙩𝙝𝙚𝙣
    for<'a> &'a Aligned1:
            TransmuteInto<&'a Aligned8>
```

Next, recall the definitions of `PromiseTransmutableInto` and `PromiseTransmutableFrom` for `&T`:
```rust
unsafe impl<'a, T> PromiseTransmutableFrom for &'a T
where
    T: PromiseTransmutableFrom
{
    type Archetype = &'a T::Archetype;
}

unsafe impl<'a, T> PromiseTransmutableInto for &'a T
where
    T: PromiseTransmutableInto
{
    type Archetype = &'a T::Archetype;
}
```
We apply these definitions to evaluate `<&_ as PromiseTransmutableInto>::Archetype` and `<&_ as PromiseTransmutableFrom>::Archetype`:
```rust
𝙞𝙛
    &'a <Aligned1 as PromiseTransmutableInto>::Archetype:
        TransmuteInto<
            &'a <Aligned8 as PromiseTransmutableFrom>::Archetype,
            NeglectStability
        >
𝙩𝙝𝙚𝙣
    for<'a> &'a Aligned1:
            TransmuteInto<&'a Aligned8>
```

Finally, recall the definitions of `PromiseTransmutableInto` for `Aligned1` and `PromiseTransmutableFrom` for `Aligned8`:
```rust
unsafe impl PromiseTransmutableInto for Aligned1 {
    type Archetype = Aligned8;
}

unsafe impl PromiseTransmutableFrom for Aligned8 {
    type Archetype = Aligned1;
}
```
We apply these definitions to evaluate `<Aligned1 as PromiseTransmutableInto>::Archetype` and `<Aligned8 as PromiseTransmutableFrom>::Archetype`:
```rust
𝙞𝙛
    &'a Aligned8:
        TransmuteInto<
            &'a Aligned1,
            NeglectStability
        >
𝙩𝙝𝙚𝙣
    for<'a> &'a Aligned1:
            TransmuteInto<&'a Aligned8>
```
Since `&'a Aligned8` can be safely transmuted into `&'a Aligned1`, the compiler accepts the transmutation of `&'a Aligned1` into `&'a Aligned1`!

</details>

---

This safety hazard can be prevented by rewriting our `where` bound like so: 
```rust
pub unsafe trait PromiseTransmutableFrom
where
    Self::Archetype: TransmuteInto<Self, NeglectStability>,
    for<'a> &'a Self::Archetype: TransmuteInto<&'a Self, NeglectStability>
{
    type Archetype;
}

pub unsafe trait PromiseTransmutableInto
where
    Self::Archetype: TransmuteFrom<Self, NeglectStability>,
    for<'a> &'a Self::Archetype: TransmuteFrom<&'a Self, NeglectStability>
{
    type Archetype;
}
```

Our previous attempt to skirt safety now produces a compiler error:
```rust
unsafe impl PromiseTransmutableInto for Aligned1 {
    type Archetype = Aligned8;
                  // ^^^^^^^^ the trait `TransmuteFrom<&'_ Aligned1, NeglectStability>` is not implemented for `&'_ Aligned8`
}
```


#### Forbidding Archetypes with Nonsense Validity Under Mutate-able Reference Transmutation
Unfortunately, the above definition is still flawed. Consider these stability declarations:
```rust
#[repr(u8)]
enum Boolean {
    False = 0,
    True = 1,
}

unsafe impl PromiseTransmutableInto for Boolean {
    type Type = u8;
}
```

Given these declarations, Rust would accept the bound `&mut Boolean: TransmuteInto<&mut u8>`. This is problematic: transmutations involving mutate-able references must not expand bit-validity.

---

<details style="background-color:#d9edf7;padding:1em">
<summary>Why would this transmutation be accepted? (Click to Expand)</summary>

Why would this transmutation be accepted? Let's solve this trait bound step-by-step:
```rust
/// 0. Initial equation:
for<'a> &'a mut Boolean:
        TransmuteInto<&'a mut u8>
```

As before, we first apply the stability rule to our equation:
```rust
𝙞𝙛
    for<'a> <&'a mut Boolean as PromiseTransmutableInto>::Archetype:
            TransmuteInto<
                <&'a mut u8 as PromiseTransmutableFrom>::Archetype,
                NeglectStability
            >
𝙩𝙝𝙚𝙣
    for<'a> '&mut Boolean:
            TransmuteInto<&'a mut u8>
```

Next, recall the definitions of `PromiseTransmutableInto` and `PromiseTransmutableFrom` for `&mut T`:
```rust
unsafe impl<'a, T> PromiseTransmutableFrom for &'a mut T
where
    T: PromiseTransmutableFrom
{
    type Archetype = &'a T::Archetype;
}

unsafe impl<'a, T> PromiseTransmutableInto for &'a mut T
where
    T: PromiseTransmutableInto
{
    type Archetype = &'a T::Archetype;
}
```
We apply these definitions to evaluate `<& mut _ as PromiseTransmutableInto>::Archetype` and `<& mut _ as PromiseTransmutableFrom>::Archetype`:
```rust
𝙞𝙛
    for<'a> &'a mut <Boolean as PromiseTransmutableInto>::Archetype:
            TransmuteInto<
                &'a mut <u8 as PromiseTransmutableFrom>::Archetype,
                NeglectStability
            >
𝙩𝙝𝙚𝙣
    for<'a> '&mut Boolean:
            TransmuteInto<&'a mut u8>
```

Finally, recall the definitions of `PromiseTransmutableInto` for `Boolean` and `PromiseTransmutableFrom` for `u8`:
```rust
unsafe impl PromiseTransmutableInto for Boolean {
    type Archetype = u8;
}

unsafe impl PromiseTransmutableFrom for u8 {
    type Archetype = Self;
}
```
We apply these definitions to evaluate `<Boolean as PromiseTransmutableInto>::Archetype` and `<u8 as PromiseTransmutableFrom>::Archetype`:
```rust
𝙞𝙛
    for<'a> &'a mut u8:
            TransmuteInto<
                &'a mut u8,
                NeglectStability
            >
𝙩𝙝𝙚𝙣
    for<'a> '&mut Boolean:
            TransmuteInto<&'a mut u8>
```
Since `&'a mut u8` can be safely transmuted into `&'a mut u8`, the compiler accepts the transmutation of `&mut Boolean` into `&mut u8`.

</details>

---

This safety hazard can be prevented by rewriting our `where` bounds like so: 
```rust
pub trait PromiseTransmutableFrom
where
    Self::Archetype: TransmuteInto<Self, NeglectStability>,
    for<'a> &'a Self::Archetype: TransmuteInto<&'a Self, NeglectStability>,
    for<'a> &'a mut Self::Archetype: TransmuteInto<&'a mut Self, NeglectStability>
{
    type Archetype;
}

pub trait PromiseTransmutableInto
where
    Self::Archetype: TransmutFrom<Self, NeglectStability>,
    for<'a> &'a Self::Archetype: TransmuteFrom<&'a Self, NeglectStability>,
    for<'a> &'a mut Self::Archetype: TransmuteFrom<&'a mut Self, NeglectStability>
{
    type Archetype;
}
```

Our previous attempt to skirt safety now produces a compiler error:
```rust
impl PromiseTransmutableInto for Boolean {
    type Archetype = u8;
                  // ^^ the trait `TransmuteFrom<&'_ mut u8, NeglectStability>` is not implemented for `&'_ mut Boolean`
}
```

### Part 2: Towards Flexibility
In the previous two sections, we bypassed the safety effects of the `PromiseTransmutable{From,Into}` traits' `where` bounds by defining archetypes whose quirks were only apparent under a *more* restrictive transmutation than that enforced by the `where` bound.

We cannot play the same trick to bypass safety with our latest `where` bound. Between transmutations involving values, shared references and unique references, the rules governing transmutations involving unique references are the most restrictive.

However, this safety comes at the expense of flexibility. Recall our last 'dangerous' archetype:
```rust
impl PromiseTransmutableInto for Boolean {
    type Archetype = u8;
                  // ^^ the trait `TransmuteFrom<&'_ mut u8, NeglectStability>` is not implemented for `&'_ mut Boolean`
}
```
This archetype is sensible in the context of value transmutations. It's sensible in the context of immutable reference transmutations. The *only* context in which it is *not* sensible are mutate-able reference transmutations! Our changes achieved safety at the significant expense of flexibility.

In this section, we begin at our previous value-safe definition of the stability declaration traits:
```rust
pub unsafe trait PromiseTransmutableFrom
where
    Self::Archetype: TransmuteInto<Self, NeglectStability>,
{
    type Archetype;
}

pub unsafe trait PromiseTransmutableInto
where
    Self::Archetype: TransmuteFrom<Self, NeglectStability>,
{
    type Archetype;
}
```
...and solve its reference-unsafety in a manner that preserves flexibility.

#### Compartmentalizing Shared Reference Transmutation Stability
Reconsider these stability declarations:
```rust
#[derive(PromiseTransmutableFrom)]
#[repr(C)]
pub struct Aligned1;

#[derive(PromiseTransmutableInto)]
#[repr(C, align(8))]
pub struct Aligned8;

unsafe impl PromiseTransmutableInto for Aligned1 {
    type Archetype = Aligned8;
}

unsafe impl PromiseTransmutableFrom for Aligned8 {
    type Archetype = Aligned1;
}
```

Given these declarations, Rust would accept the bound `&Aligned1: TransmuteInto<&Aligned8>`. This is problematic: `Aligned8` has *stricter* alignment requirements than `Aligned1`.

In our derivation of why the compiler accepted this transmutation, we referred to the stability declarations of `&T`:
```rust
impl<'a, T> PromiseTransmutableFrom for &'a T
where
    T: PromiseTransmutableFrom,
{
    type Archetype = &'a T::Archetype;
}

impl<'a, T> PromiseTransmutableInto for &'a T
where
    T: PromiseTransmutableInto
{
    type Archetype = &'a T::Archetype;
}
```
Rather than change the `where` bound of the *definitions* of `PromiseTransmutableFrom` and `PromiseTransmutableInto`, we can simply change the `where` bound of those two *implementations* of the traits:
```rust
impl<'a, T> PromiseTransmutableFrom for &'a T
where
    T: PromiseTransmutableFrom,
    for<'a> &'a Self::Archetype: TransmuteInto<&'a Self, NeglectStability>
{
    type Archetype = &'a T::Archetype;
}

impl<'a, T> PromiseTransmutableInto for &'a T
where
    T: PromiseTransmutableInto,
    for<'a> &'a Self::Archetype: TransmuteFrom<&'a Self, NeglectStability>
{
    type Archetype = &'a T::Archetype;
}
```

#### Compartmentalizing Unique Reference Transmutation Stability
Likewise, for unique references, we go from this:
```rust
impl<'a, T> PromiseTransmutableFrom for &'a mut T
where
    T: PromiseTransmutableFrom
{
    type Archetype = &'a mut T::Archetype;
}

impl<'a, T> PromiseTransmutableInto for &'a mut T
where
    T: PromiseTransmutableInto
{
    type Archetype = &'a mut T::Archetype;
}
```
...to this:
```rust
impl<'a, T> PromiseTransmutableFrom for &'a mut T
where
    T: PromiseTransmutableFrom,
    for<'a> &'a mut Self::Archetype: TransmuteInto<&'a mut Self, NeglectStability>
{
    type Archetype = &'a T::Archetype;
}

impl<'a, T> PromiseTransmutableInto for &'a mut T
where
    T: PromiseTransmutableInto,
    for<'a> &'a mut Self::Archetype: TransmuteFrom<&'a mut Self, NeglectStability>
{
    type Archetype = &'a T::Archetype;
}
```

### Part 3: Towards Sensibility

In the previous two parts, we developed safe-and-flexible stability declaration traits. While we cannot make unsafe archetype choices, we can still make *silly* archetype choices. For instance, we can declare the stability of our types in terms of types that are *not* stable:

```rust
#[repr(u8)]
pub struct Foo { ... }

#[repr(u8)]
pub struct Bar { ... }

impl PromiseTransmutableInto for Foo {
    type Archetype = Bar;
}
```

We can prevent these sorts of silly archetypes by requiring that `Archetype` itself is stable, and that its stability archetype is not transitively defined:

```rust
pub trait PromiseTransmutableFrom
{
    type Archetype
        : TransmuteInto<Self, NeglectStability>
        + PromiseTransmutableFrom;
}

pub trait PromiseTransmutableInto
{
    type Archetype
        : TransmuteFrom<Self, NeglectStability>
        + PromiseTransmutableInto;
}
```

# Drawbacks
[drawbacks]: #drawbacks

## No Notion of Platform Stability
The stability declaration traits communicate library layout stability, but not *platform* layout stability. A transmutation is platform-stable if it compiling one one platform implies it will compile on all other platforms. Unfortunately, platform-unstable types are common;  e.g.:

- All pointer-related primitive types (`usize`, `isize`, `*const T`, `*mut T`, `&T`, `&mut T`) possess platform-dependent layouts; their sizes and alignments are well-defined, but vary between platforms. Concretely, whether `usize` is `TransmuteInto<[u8; 4]>` or `TransmuteInto<[u8; 8]>` will depend on  the platform.
- The very existence of some types depends on platform, too; e.g., the contents of [`core::arch`](https://doc.rust-lang.org/stable/core/arch/), [`std::os`](https://doc.rust-lang.org/stable/std/os/), and [`core::sync::atomic`](https://doc.rust-lang.org/stable/std/sync/atomic/) all depend on platform.

Our proposed stability system is oblivious to the inter-platform variations of these types. Expanding our stability system to be aware of inter-platform variations would introduce considerable additional complexity:
<ol>
<li>

**Cognitive Complexity:** For types whose layout varies between platforms, the [stability] declaration traits could, *perhaps*, be adapted to encode platform-related guarantees. We anticipate this would contribute substantial cognitive complexity. Type authors, even those with no interest in cross-platform stability, would nonetheless need to reason about the layout properties of their types on platforms that might not yet exist.
</li>
<li>

**Ergonomic Complexity:** Platform instabilities are contagious: a type that *contains* a platform-unstable type is, itself, platform-unstable. Due to the sheer virulence of types with platform-dependent layouts, an explicit '`NeglectPlatformStability`' option would need to be used for *many* simple transmutations. The ergonomic cost of this would also be substantial.

</li>
<li>

**Implementation Complexity:** The mechanisms proposed by this RFC are, fundamentally, applications of and additions to Rust's type system (i.e., they're traits). Mechanisms that impact platform stability, namely `#[cfg(...)]` annotations, long precede type-resolution and layout computation in the compilation process. For instance, it's possible to define types with impossible layouts:    
```rust
#[cfg(any())]
struct Recursive(Recursive);
```
This program compiles successfully on all platforms because, from the perspective of later compilation stages, `Recursive` may as well not exist.

</li>
</ol>

The issues of platform layout stability exposed by this RFC are not fundamentally different from the challenges of platform API stability. These challenges are already competently addressed by the mechanisms proposed in [RFC1868](https://github.com/rust-lang/rfcs/pull/1868). For this reason, and for the aforementioned concerns of additional complexity, we argue that communicating and enforcing platform layout stability must remain outside the scope of this RFC.

## Stability of *Unsafe* Transmutations
[drawback-unsafe-stability]: #Stability-of-Unsafe-Transmutations

The model of stability proposed by this RFC frames stability as a quality of *safe* transmutations. A type author cannot specify stability archetypes for *unsafe* transmutations, and it is reasonable to want to do so.

To accomodate this, we may modify the definitions of `PromiseTransmutableFrom` and `PromiseTransmutableInto` to consume an optional `Neglect` parameter, to allow for stability declarations for unsafe transmutations:
```rust
pub trait PromiseTransmutableFrom<Neglect = ()>
where
    Neglect: UnsafeTransmuteOptions
{
    type Archetype
        : TransmuteInto<Self, Sum<Neglect, NeglectStability>>
        + PromiseTransmutableInto<Sum<Neglect, NeglectStability>>;
}

pub trait PromiseTransmutableInto<Neglect = ()>
where
    Neglect: UnsafeTransmuteOptions
{
    type Archetype
        : TransmuteFrom<Self, Sum<Neglect, NeglectStability>>
        + PromiseTransmutableInto<Sum<Neglect, NeglectStability>>;
}
```
Implementations of these traits for a given `Neglect` declares that a transmutation which is accepted while neglecting a particular set of checks (namely the set encoded by `Neglect`) will *continue* to be possible.

We omit these definition from this RFC's recommendations because they are not completely satisfying. For instance, `Neglect` is a *logically* unordered set of options, but is encoded as a tuple (which *is* ordered). To declare a transmutation that requires neglecting validity and alignment checks as stable, only *one* of these impls ought to be necessary:

```rust
impl PromiseTransmutableFrom<(NeglectAlignment, NeglectValidity)> for Foo
{
    ...
}

impl PromiseTransmutableFrom<(NeglectValidity, NeglectAlignment)> for Foo
{
    ...
}
```
Writing *both* impls (as we do above) is logically nonsense, but is nonetheless supported by Rust's coherence rules.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Case Studies

### Case Study: Making Unsafe Rust Safer
[case-study-safer-unsafe]: #Case-Study-Making-Unsafe-Rust-Safer

This RFC proposes mechanisms for safe transmutation and safer *unsafe* transmutation. How might [Kelley's motivating example][motivation-safer-unsafe] change in light of these mechanisms?

#### Safer Unsafe
For the compiler to accept a use of `unsafe_transmute`, we must:
   - use `unsafe_transmute` with the `NeglectAlignment` option
   - annotate `Foo` with `#[repr(C)]` so its layout is defined
   - annotate `Foo` with `#[derive(PromiseTransmutableFrom,PromiseTransmutableInto)]` to indicate its layout is library-stable (alternatively, we could also use the  `NeglectStability` option)

The sum of these changes:
```rust
use core::transmute::{
    unsafe_transmute,
    options::NeglectAlignment,
    stability::FixedLayout,
};


#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(C)]
pub struct Foo {
    a: i32,
    b: i32,
}

fn main() {
    let mut array: [u8; 1024] = [1; 1024];

    let mut foo = unsafe {
        // check alignment ourselves
        assert_eq!((src as *const u8 as usize) % align_of::<Foo>(), 0);

        // transmute `&mut array` to `&mut Foo`
        // permitted becasue we promise to check the alignment ourselves
        let foo = unsafe_transmute::<&mut [u8; 1024], &mut Foo, NeglectAlignment>(&mut array);
    };

    foo.a += 1;
}
```

#### No Unsafe Needed!
Better yet, we may use a completely *safe* transmute if we statically align the byte array:

```rust
use core::transmute::{
    TransmuteInto,
    options::NeglectAlignment,
    stability::FixedLayout,
};


#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(C)]
struct Foo {
    a: i32,
    b: i32,
}

#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(C, align(4)]
struct Aligned([u8; 1024]);

fn main() {
    let mut array: Aligned = Aligned([1; 1024]);

    let foo : &mut Foo = (&mut array[0]).transmute_into();

    foo.a += 1;
}
```

### Case Study: Abstractions for Packet Parsing
***TODO: josh***

### Case Study: Abstractions for Pointer Bitpacking
[case-study-alignment]: #case-study-abstractions-for-pointer-bitpacking

A [previous motivating example](#expressing-layout-requirements) involved a generic abstraction only usable with types meeting certain layout requirements: pointer bitpacking. Using this RFC's mechanisms, we can require that a reference to a generic type `T` has alignment of at least a given value (e.g., `8`) by first defining a ZST with that alignment:
```rust
#[derive(PromiseTransmutableFrom)]
#[repr(align(8)]
struct Aligned8;
```
and then using this `where` bound:
```rust
where
    &T: TransmuteInto<&Aligned8>
```

## Rationale: `TransmuteFrom`/`TransmuteInto`

### Why support arbitrary transmutation?
Some [prior art][prior-art], especially in the crate ecosystem, provides an API that [only supports transmutations involving particular types](#Source-and-Destination-Types-Supported) (e.g., from/into bytes). As we discuss in the [prior art][prior-art] section, we believe that the inflexibility of such approaches make them a poor basis of a language proposal. In particular, these restrictive approaches don't leave room to grow: supporting additional transmutations requires additional traits.

The API advocated by this proposal is unopinionated about what transmutations users might wish to do, and what transmutations the compiler is able to reason about. The implementation of this RFC may be initially very simple (and perhaps support no more than the restrictive approaches allow for), but then subsequently grow in sophistication—*without* necessitating public API changes.

### Why *two* traits?
If `TransmuteInto` is implemented in terms of `TransmuteFrom`, why provide it at all? We do so for consistency with libcore's [`From`/`Into`](https://doc.rust-lang.org/stable/rust-by-example/conversion/from_into.html) traits, and because directionality conveys intent: `TransmuteFrom` connotes *conversion*, whereas `TransmuteInto` connotes initialization. We believe that the supporting code examples of this RFC demonstrate the explanatory benefits of providing *both* traits.

## Rationale: Transmutation Options

### Granularity
Although the focus of our API is statically-correct, infalible transmutations, the ability to opt-out of particular static checks is essential for building safer *fallible* mechanisms, such as alignment-fallible [reference casting][ext-ref-casting], or validity-fallible transmutations (e.g., `bool` to `u8`).

### Representation
Although transmutations options exist at a type-level, they're represented as type-level tuples, whose familiar syntax is identical to value-level tuples. An empty tuple seems like the natural choice for encoding *don't neglect anything*.

We could not identify any advantages to representing options with const-generics. There is no clear syntactic advantage: tuples remain the most natural way to encode ad-hoc products of items. The comparative lack of default values for const-generic parameters poses an ergonomic *disadvantage*.


## Rationale: Stability
### Distinguishing Stability from Possibility
[Prior art][mechanism-manual] which requires an author's manual implementation of conversion traits conflates stability with possibility; i.e., there may be types for which a transmutation is known to be valid for which the authors do not implement the required traits for safe transmutation. For such types, transmuters must fall all the way back to the *wildly* unsafe `mem::transmute`; there is no middle-ground where transmuters can retain all static guarantees *besides* stability. By distinguishing possibility from stability, our proposal provides this middle ground.

### Distinguishing Stability from `#[repr(C)]`
In our proposal, `#[repr(C)]` does not connote any promises of layout stability for SemVer purposes. It's [been suggested](https://rust-lang.zulipchat.com/#narrow/stream/216762-project-safe-transmute/topic/RFC.3A.20Stability.20Declaration.20Traits/near/204011238) that the presence of `#[repr(C)]` *already* connotes total layout stability; if so, explicit stability declaration traits are superfluous. However, we are unaware of any documentation indicating that `#[repr(C)]` carries this implication. (It's also [been suggested](https://rust-lang.zulipchat.com/#narrow/stream/216762-project-safe-transmute/topic/typic/near/201165897) that `#[repr(C)]` should *not* connote stability, because that would pose a stability hazard. We agree with this assessment.)

### Permitting Granular Stability Declarations
The [stability declaration traits][stability] permit granular and incomplete promises of layout stability (e.g., guaranteeing the size and validity qualities of a type, but *not* its alignment. An alternative formulation of stability might be all-or-nothing (i.e., equivalent to only being able to write `Archetype = Self`). However, members of the safe-transmute working group have [expressed](https://rust-lang.zulipchat.com/#narrow/stream/216762-project-safe-transmute/topic/Transmutability.20Intrinsic/near/202712834) an interest in granular stability declarations.

Our proposed API supports complex, granular stability declarations, all-the-while retaining simplicity in simple use-cases (namely, one can simply the stability declaration traits). 


## Alternative: Implementing this RFC in a Crate

This RFC builds on ample [prior art][prior-art] in the crate ecosystem, but these efforts strain against the fundamental limitations of crates. Fundamentally, safe transmutation efforts use traits to expose layout information to the type system. The burden of ensuring safety [is usually either placed entirely on the end-user, or assumed by complex, incomplete proc-macro `derives`][mechanism-manual].

An exception to this rule is the [typic][crate-typic] crate, which utilizes complex, type-level programming to emulate a compiler-supported, "smart" `TransmuteFrom` trait (like the one proposed in this RFC). Nonetheless, [typic][crate-typic] is fundamentally limited: since Rust does not provide a type-level mechanism for reflecting over the structure of arbitrary types, even [typic][crate-typic] cannot judge the safety of a transmutation without special user-added annotations on type definitions. Although [typic][crate-typic] succeeds as a proof-of-concept, its maintainability is questionable, and the error messages it produces are [lovecraftian](https://en.wikipedia.org/wiki/Lovecraftian_horror).

The development approaches like [typic][crate-typic]'s could, perhaps, be eased by stabilizing [frunk](https://crates.io/crates/frunk)-like structural reflection, or (better yet) by stabilizing a compiler plugin API for registering "smart" traits like `TransmuteFrom`. However, we suspect that such features would be drastically harder to design and stabilize. 

Regardless of approach, almost all [prior art][prior-art] attempts to reproduce knowledge *already* possessed by rustc during the compilation process (i.e., the layout qualities of a concrete type). Emulating the process of layout computation to any degree is an error-prone duplication of effort between rustc and the crate, in a domain where correctness is crucial.

Finally, community-led, crate-based approaches are, inescapably, unauthoritative. These approaches are incapable of fulfilling our motivating goal of providing a *standard* mechanism for programmers to statically ensure that a transmutation is safe, sound, or stable.

# Prior art
[prior-art]: #prior-art

[crate-plain]: https://crates.io/crates/plain
[crate-bytemuck]: https://crates.io/crates/bytemuck
[crate-dataview]: https://crates.io/crates/dataview
[crate-safe-transmute]: https://crates.io/crates/safe-transmute
[crate-pod]: https://crates.io/crates/pod
[crate-uncon]: https://crates.io/crates/uncon
[crate-typic]: https://crates.io/crates/typic
[crate-zerocopy]: https://crates.io/crates/zerocopy
[crate-convute]: https://crates.io/crates/convute
[crate-byterepr]: https://crates.io/crates/byterepr

[2017-02]: https://internals.rust-lang.org/t/pre-rfc-safe-coercions/4823
[2018-03]: https://internals.rust-lang.org/t/pre-rfc-frombits-intobits/7071
[2018-03-18]: https://internals.rust-lang.org/t/pre-rfc-frombits-intobits/7071/23
[2018-05-18]: https://internals.rust-lang.org/t/pre-rfc-trait-for-deserializing-untrusted-input/7519
[2018-05-23]: https://github.com/joshlf/rfcs/blob/joshlf/from-bytes/text/0000-from-bytes.md
[2019-09]: https://internals.rust-lang.org/t/specifying-a-set-of-transmutes-from-struct-t-to-struct-u-which-are-not-ub/10917
[2019-11]: https://internals.rust-lang.org/t/pre-rfc-safe-transmute/11347
[2019-12-05-gnzlbg]: https://gist.github.com/gnzlbg/4ee5a49cc3053d8d20fddb04bc546000
[2019-12-05-v2]: https://internals.rust-lang.org/t/pre-rfc-v2-safe-transmute/11431
[2020-07]: https://internals.rust-lang.org/t/pre-rfc-explicit-opt-in-oibit-for-truly-pod-data-and-safe-transmutes/2361


## Dimensions of Variation

### Conversion Complexity
Prior work differs in whether it supports complex conversions, or only simple transmutation. [*Pre-RFC FromBits/IntoBits*][2018-03]'s proposed traits include conversion methods that are implemented by type authors. Because end-users provide their own definitions of these methods, they can be defined to do more than just transmutation (e.g., slice casting). (This approach is similar to the [uncon][crate-uncon] crate's [`FromUnchecked`](https://docs.rs/uncon/1.*/uncon/trait.FromUnchecked.html) and [`IntoUnchecked`](https://docs.rs/uncon/1.*/uncon/trait.IntoUnchecked.html) traits, which provide unsafe conversions between types. These traits are safe to implement, but their conversion methods are not.)

In contrast, our RFC focuses only on transmutation. Our `TransmutableFrom` and `TransmutableInto` traits serve as both a marker *and* a mechanism: if `Dst: TransmuteFrom<Src>`, it is sound to transmute from `Dst` into `Src` using `mem::transmute`. However, these traits *also* provide transmutation methods that are guaranteed to compile into nothing more complex than a `memcpy`. These methods cannot be overridden by end-users to implement more complex behavior. 

The signal and transmutability and mechanism are, in principle, separable. The [convute][crate-convute] crate's [`Transmute<T>`](https://docs.rs/convute/0.2.0/convute/marker/trait.Transmute.html) trait is an unsafe marker trait representing types that can be transmuted into `T`. This is *just* a marker trait; the actual conversion mechanisms are provided by a [separate suite](https://docs.rs/convute/0.2.0/convute/convert/index.html) of traits and functions. Our RFC combines marker with mechanism because we feel that separating these aspects introduces additional complexity with little added value. 

### Conversion Fallibility
Prior work differs in whether it supports only infallible conversions, or fallible conversions, too. The [convute][crate-convute] crate's [`TryTransmute<T>`](https://docs.rs/convute/0.2.0/convute/marker/trait.TryTransmute.html) trait provides a method, `can_transmute`, that returns true a transmutation from `Self` to `T` is valid for a particular value of `&self`. An early version of [typic][crate-typic] abstracted a similar mechanism into an [`Invariants`](https://docs.rs/typic/0.1.0/typic/transmute/trait.Invariants.html) trait, with additional facilities for error reporting. [*Draft-RFC: `Compatible`/`TryCompatible`*][2019-12-05-gnzlbg] employs a similar mechanism to typic.

Typic removed support for fallible transmutation after reckoning with several challenges:
- The causes of uncertain failure could be languaged-imposed (e.g., alignment or validity requirements), or library imposed (i.e., invariants placed on a structure's private fields).
- The points of uncertain failures could be arbitrarily 'deep' into the fields of a type.
- Error reporting incurs a runtime cost commensurate with the detail of the reporting, but the detail of reporting required by end-user depends on use-case, not just type. For instance: for some use-cases it may be necessary to know where and why a byte was not a valid `bool`; in others it may be sufficient to know simply *whether* an error occurred.

Finally, we observed that the mechanisms of fallible transmutation were basically separable from the mechanisms of infallible transmutation, and thus these challenges could be addressed at a later date. For these reasons, our RFC *only* addresses infallible transmutation.

While this RFC does not provide a grand, all-encompassing mechanism for fallible transmutation, the fundamental mechanisms of our RFC are useful for constructing safer, purpose-built fallible conversion mechanisms; e.g.:
```rust
/// Try to convert a `&T` into `&U`.
///
/// This produces `None` if the referent isn't appropriately
/// aligned, as required by the destination type.
pub fn try_cast_ref<'t, 'u, T, U>(src: &'t T) -> Option<&'u U>
where
    &'t T: TransmuteInto<&'u U, NeglectAlignment>,
{
    if (src as *const T as usize) % align_of::<U>() != 0 {
        None
    } else {
        // Safe because we dynamically enforce the alignment
        // requirement, whose static check we chose to neglect.
        Some(unsafe { src.unsafe_transmute_into() })
    }
}
```
In this approach, our RFC is joined by crates such as [plain](https://docs.rs/plain/0.2.3/plain/#functions), [bytemuck](https://docs.rs/bytemuck/1.*/bytemuck/#functions), [dataview](https://docs.rs/dataview/0.1.1/dataview/struct.DataView.html#methods), [safe-transmute](https://docs.rs/safe-transmute/0.11.0/safe_transmute/fn.transmute_one.html), [zerocopy](https://docs.rs/zerocopy/0.3.0/zerocopy/struct.LayoutVerified.html#methods), and [byterepr](https://docs.rs/byterepr/0.1.0/byterepr/trait.ByteRepr.html#provided-methods), and several pre-RFCs (such as [this][2018-05-18] and [this](https://github.com/joshlf/rfcs/blob/joshlf/from-bits/text/0000-from-bits.md#library-functions)). The ubiquity of these mechanisms makes a strong case for their inclusion in libcore.

### Source and Destination Types Supported
Prior work differs in whether its API surface is flexible enough to support transmutation between arbitrary types, or something less.

#### Arbitrary Types
Approaches supporting transmutations between arbitrary types invariably define traits akin to: 
```rust
/// Indicates that `Self` may be transmuted into `Dst`.
pub unsafe trait TransmuteInto<Dst>
{ ... }

/// Indicates that `Self` may be transmuted from `Dst`.
pub unsafe trait TransmuteFrom<Src>
{ ... }
```
This approach, taken by our RFC, is used by at least two crates:
- The [convute][crate-convute] crate's [`Transmute<T>`](https://docs.rs/convute/0.2.0/convute/marker/trait.Transmute.html) trait is akin to the above definition of `TransmuteInto`.
- The [typic][crate-typic] crate's [`TransmuteInto`](https://docs.rs/typic/0.3.0/typic/transmute/trait.TransmuteInto.html) and [`TransmuteFrom`](https://docs.rs/typic/0.3.0/typic/transmute/trait.TransmuteFrom.html) traits almost exactly mirror the above definitions.

...and several proposals:
- [*Pre-RFC: Safe coercions*][2017-02] proposes a `Coercible<A, B>` trait that is implemented if `A` is safely transmutable into `B`.
- [*Pre-RFC: `FromBits`/`IntoBits`*][2018-03] proposes the traits `IntoBits<U>` and `FromBits<T>.`
- [*Draft-RFC: `FromBytes`*][2018-05-23] proposes the traits `IntoBytes<U>` and `FromBytes<T>.`
- [*Draft-RFC: `Compatible`/`TryCompatible`*][2019-12-05-gnzlbg] proposes the trait `Compatible<U>`, akin to the above definition of `TransmuteInto`.

##### From/Into Bytes Transmutations
Other approaches adopt an API that only supports transmutation of a type into initialized bytes, and from initialized bytes. These approaches invariably define traits akin to:
```rust
/// Indicates that a type may be transmuted into an appropriately-sized array of bytes.
pub unsafe trait IntoBytes
{}

/// Indicates that a type may be transmuted from an appropriately-sized array of bytes.
pub unsafe trait FromBytes
{}
```
This is the approach taken by the [zerocopy][crate-zerocopy] crate, and the [*Pre-RFC: Safe Transmute*][2019-11] and [*Pre-RFC: Safe Transmute v2*][2019-12-05-v2] proposals.

This approach is strictly less flexible than an API supporting transmutation between arbitrary types. It is incapable of representing transmutations of bytes into types with validity constraints, and incapable of representing transmutations of types with padding bytes into bytes.

Supporting additional transmutation source and destination types requires a commensurate addition of conversion traits. For instance, some of [zerocopy][crate-zerocopy]'s users [require](https://fuchsia-review.googlesource.com/c/fuchsia/+/306036/2#message-a1a0c9cf16e3dec24e7b0548e3c09382f63783f0) a trait that reflects types which can be transmuted from a buffer of zeroed bytes. This would require introducing an additional trait, `FromZeros`.

An advantage of this API is that it gives descriptive names to perhaps the two most common transmutations. However, an API providing transmutation between arbitrary types can encode `FromBytes` and `IntoBytes`:
```rust
// `Dst` is `FromBytes` if it can be safely transmuted *from* an
// equivalently sized array of `u8`.
unsafe impl<Dst> FromBytes for Dst
where
    Dst: TransmuteFrom<[u8; size_of::<Dst>()]>,
{}

// `Src` is `IntoBytes` if it can be safely transmuted *into* an
// equivalently sized array of `u8`.
unsafe impl<Src> IntoBytes for Src
where
    Src: TransmuteInto<[u8; size_of::<Src>()]>,
{}
```
For these reasons, we argue that a `FromBytes`/`ToBytes` style API is a poor foundation for in-language safe transmutation.

##### Bytes-to-Bytes Transmutations (aka "Plain Old Data")
Finally, many approaches (especially crates) supply a marker trait that represents "plain old data"; e.g.:
```rust
/// Implemented by types that are "plain old data":
pub unsafe trait PlainOldData
{}
```
This sort of trait is present in crates such as [plain](https://docs.rs/plain/0.2.3/plain/trait.Plain.html), [bytemuck](https://docs.rs/bytemuck/1.*/bytemuck/trait.Pod.html), [dataview](https://docs.rs/dataview/0.1.1/dataview/trait.Pod.html), [safe-transmute](https://docs.rs/safe-transmute/0.11.0/safe_transmute/trivial/trait.TriviallyTransmutable.html), and [pod](https://docs.rs/pod/0.5.0/pod/trait.Pod.html), and at least two language proposals ([here][2018-05-18] and [here][2020-07]).

The exact definition of what constitutes "plain old data" varies between crates. One simple definition is that a type `T` is "plain old data" if it can be transmuted both from and into initialized bytes; i.e.:
```rust
unsafe impl<T> PlainOldData for T
where
    T: FromBytes + IntoBytes,
{}
```

This definition precludes useful transmutations. For instance, `MaybeUninit<u8>` is transmutable from a `u8`, but not *into* a `u8`.

Given this inflexibility, we argue that this approach is a poor foundation for in-language safe transmutation.


### Implementation Mechanism
Not only does prior work differ in which traits are used to encode valid transmutations, they differ in the level of user intervention required to take advantage of the traits. 

#### Manual
[mechanism-manual]: #Manual
Fully manual approaches require type authors to implement the transmutation traits manually. The involved traits are `unsafe`, so it is up to type authors to verify for themselves that their hand-written implementations are sound. This is the approach taken by crates such as [plain][crate-plain], [bytemuck][crate-bytemuck], [safe-transmute][crate-safe-transmute], and [pod][crate-pod], and at least one language proposal: [*Pre-RFC: Safe Transmute*][2019-12-05-v2] (which advocates for a "plain old data" API).

In semi-manual approaches, type authors simply `derive` the applicable traits, using `derive` macros that produce a compile-error if the implementation is not sound. This approach is realized by crates such as ([zerocopy](https://docs.rs/zerocopy/0.3.0/zerocopy/#derives), [zeroable](https://docs.rs/zeroable/0.2.0/zeroable/) and [dataview](https://docs.rs/dataview/0.1.1/dataview/derive.Pod.html)) and advocated by at least two language proposals: [*Pre-RFC: Safe Transmute v2*][2019-12-05-v2] (which advocates for a `FromBytes`/`IntoBytes`-style API), and [*Pre-RFC FromBits/IntoBits*][2018-03] (which advocates for a general-transmutation API).

We believe that the implementation burden these approaches place on end-users, and their inflexibility, make them a poor foundation for in-language safe transmutation:
- These approaches require authors to implement and, potentially, verify a large number of `unsafe` traits, ranging from *O(n)* implementations for plain-old-data trait approaches, to potentially [*many* more](https://internals.rust-lang.org/t/pre-rfc-frombits-intobits/7071/28).
- These approaches are generally impractical for APIs that permit truly general transmutation, as type authors can only construct implementations of the transmutation traits for types they have at their disposal.
- These approaches conflate transmutation stability with transmutation safety. An end-user wishing to transmute a type for which its author has *not* manually implemented the applicable traits must resort to the wildly unsafe `mem::transmute`.



#### Automatic
Automatic approaches implement the transmutation traits without user intervention, whenever it is sound to do so. This is the approach taken by our RFC. Automatic mechanisms appear in at least four prior language proposals:
- [*Pre-RFC: Safe coercions*][2017-02]
- [*Draft-RFC: `from_bytes`*][2018-05-23]
- [*Pre-RFC: Trait for deserializing untrusted input*][2018-05-18]
- [*Draft-RFC: `compatible_trait`*][2019-12-05-gnzlbg]

The [typic][crate-typic] crate mocks a fully-automatic approach: its `TransmuteFrom` trait is usable with any types that are `repr(C)`, or otherwise have a well-defined memory layout. (In practice, since Rust lacks reflection over type definitions, `repr(C)` annotations much be changed to `typic::repr(C)`.)

### Stability Hazards
Fully automatic approaches introduce, at the very least, a stability hazard: they supply a safe constructor for types, without the consent of those types' authors. If a type author hid the internals of their type because they do not wish for its implementation details to become a part of the type's API for SemVer purposes, an automatic transmutation mechanism subverts that intent.

No attempt to avoid this hazard is made by most of the proposals featuring automatic mechanisms; e.g.:
- [*Draft-RFC: `from_bytes`*][2018-05-23]
- [*Pre-RFC: Trait for deserializing untrusted input*][2018-05-18]
- [*Draft-RFC: `compatible_trait`*][2019-12-05-gnzlbg]

#### Hazard-Avoidant
The automatic mechanism proposed by [*Pre-RFC: Safe coercions*][2017-02] exploits field visibility, requiring that all fields that have different types in `Src` and `Dst` are visible at the location where the coercion is made. This approach falls short in three respects:
1. Confining the visibility requirement only to fields of *different* types is insufficient; two different types with identical field types may subject those fields to different invariants. 
2. The 'location' where the coercion is made is ill-defined; the presence of the proposed `Coercible` trait may be far-removed from the location of the actual conversion (if any conversion occurs at all).
3. Field visibility stabilizes the structure of a type, but *not* its layout (e.e., its size).

Our RFC exploits the related concept of *constructability*, which is a property of a struct, or enum variant (rather than solely a property of fields). However, we recognize that it may be difficult to test for constructability within the trait resolution process.

The simplified definition of *constructability* we propose is the same employed by [typic][crate-typic] (which uses the term "visibility"). [Typic][crate-typic] regards the pub-in-prive soundness hole of the simplified definition to be sufficiently niche that `NeglectStability` remains "safe". However, unlike [typic][crate-typic], we believe that this simplified definition imposes a safety hazard substantial enough to warrant making `NeglectStability` initially an unsafe transmute option. 

Our RFC separates *constructability*, which concerns what aspects of a type's structure are part of its public API, and *stability*, which concerns the aspects of a type's layout that are part of its public API for SemVer purposes. This distinction does not appear in prior work.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### Questions To Be Resolved Before RFC Acceptance
The following unresolved questions should be resolved during the RFC process:

##### Unhandled Use-Cases?
We endeavored to design an API surface with ([nearly][drawbacks]) zero compromises. However, if you have a use-case that you believe is neither satisfied outright by our proposal, nor [aided][future-possibilities] by our proposal, we would *urgently* like to hear of it.

##### Extensions for Inclusion?
In [*Future Possibilities*][future-possibilities], we propose a number of additional abstractions that are aided by this RFC. Some of these abstractions are commonplace in [prior art][prior-art] and should perhaps be included with this RFC. Some of our proposed extensions could begin their crates that work on stable Rust; others, such as [generic atomics][future-possibility-generic-atomics], require nightly-only intrinsics.

### Questions To Be Resolved Before Feature Stabilization
The following unresolved questions should be resolved before feature stabilization:

##### Layout-Stability for Unsafe Transmutations?
We [observe][drawback-unsafe-stability] that our proposed model for stability declaration, although very expressive, does not permit type authors to declare the stability of *unsafe* transmutations. Alongside that observation, we suggest a [SemVer-compatible](https://github.com/rust-lang/rfcs/blob/master/text/1105-api-evolution.md#minor-change-adding-a-defaulted-type-parameter) upgrade of the stability declaration traits that may resolve this shortcoming.

While it is unclear if there is any demand for this degree of flexibility, this upgrade-path should be carefully considered *before* stabilizing (and thus committing) to this RFC's layout stability declaration traits.


### Questions Out of Scope
We consider the following unresolved questions to be out-of-scope of *this* RFC process:

##### Design of `NeglectConstructability`?
`TransmuteFrom` and `TransmuteInto` require that the destination type has a matching constructor in which all fields are marked `pub`. Conspicuously *missing* from this RFC is a `NeglectConstructability` unsafe option to disable this check.

The omission is intentional. The consequences of such an option are suprising in both their subtlety and their unsafety. Some of unsafe Rust's hairiest interactions lie at the intersections of `!Send`, `!Sync`, `UnsafeCell` and restricted field visibility. These building blocks are used to build safe, public abstractions that encapsulate unsafe, hidden internals.

# Future possibilities
[future-possibilities]: #future-possibilities

## Extension: `PromiseTransmutable` Shorthand
[extension-promisetransmutable-shorthand]: #extension-promisetransmutable-shorthand

We anticipate that *most* users will merely want to [promise][stability-common] that their types are as-stable-as-possible. To do so, this RFC proposes the shorthand:
```rust
#[derive(PromiseTransmutableFrom, PromiseTransmutableInto)]
#[repr(C)]
pub struct Foo(pub Bar, pub Baz);
```
As a shorthand, this is still rather long. For such users, the separability of `PromiseTransmutableFrom` and `PromiseTransmutableInto` is totally irrelevant. We therefore propose a `derive(PromiseTransmutable)` shorthand, such that this:
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

However, we caution *against* adding a corresponding trait or trait alias; e.g.:
```rust
trait PromiseTransmutable = PromiseTransmutableFrom + PromiseTransmutableInto;
```
The vast majority of users will *only* confront the stability declaration traits in the context of deriving them; the *only* scenario in which end-users will refer to these traits in a type-context is the [rare use-case][stability-uncommon] of *manually* implementing them. For such users, the separability of `PromiseTransmutableFrom` and `PromiseTransmutableInto` *is* relevant. The availability of a `PromiseTransmutable` trait or trait alias in this scenario would be a distraction, since referring to it in a type-context is almost certainly a misstep.

We acknowledge that it is unusual for a `derive` macro to not create an item of the same name, but this weirdness is outweighed by the weirdness of the alternative: providing a trait for which there is almost no good use. 

## Extension: Byte Transmutation Traits
[marker-traits]: #Extension-Byte-Transmutation-Traits
[future-possibility-safe-initialization]: #Safe-Initialization

The marker traits from [zerocopy][crate-zerocopy] are expressible using `TransmuteFrom` and `TransmuteInto`. In principle, these traits could remain in crates. However, we believe their clarity and broad usefulness may justify their inclusion in `libcore`.

### `IntoBytes`
Indicates that a type may be transmuted into an appropriately-sized array of bytes.
```rust
pub unsafe trait IntoBytes<Neglect = ()>
where
    Neglect: UnsafeTransmuteOptions
{}

// `Src` is `IntoBytes` if it can be safely transmuted *into* an
// equivalently sized array of `u8`.
unsafe impl<Src, Neglect> IntoBytes<Options> for Src
where
    Src: Sized + TransmuteInto<[u8; size_of::<Src>()], Neglect>,
    Neglect: UnsafeTransmuteOptions
{}
```

### `FromBytes`
Indicates that a type may be transmuted from an appropriately-sized array of bytes.
```rust
pub unsafe trait FromBytes<Neglect = ()>
where
    Neglect: UnsafeTransmuteOptions
{}

// `Dst` is `FromBytes` if it can be safely transmuted *from* an
// equivalently sized array of `u8`.
unsafe impl<Dst, Neglect> FromBytes<Options> for Dst
where
    Dst: Sized + TransmuteFrom<[u8; size_of::<Dst>()], Neglect>,
    Neglect: UnsafeTransmuteOptions
{}
```

### `FromZeros`
[future-possibility-safe-initialization]: #FromZeros

Indicates that a type may be transmuted from an appropriately-sized array of zeroed bytes. Equivalently, `FromZeros` indicates that a type may be safely initialized with `mem::zeroed`.

```rust
pub unsafe trait FromZeros<Neglect = ()>
where
    Neglect: UnsafeTransmuteOptions
{
    /// Safely initialize `Self` from zeroed bytes.
    fn zeroed() -> Self
    where
        Neglect: SafeTransmuteOptions;

    /// Unsafely initialize `Self` from zeroed bytes.
    fn unsafe_zeroed() -> Self
    where
        Neglect: SafeTransmuteOptions;
}

#[derive(Copy)]
#[repr(u8)]
enum Zero {
    Zero = 0u8
}

// `T` is `FromZeros` if it can be safely transmuted *from* an
// equivalently sized array of zeroed bytes.
unsafe impl<Dst, Neglect> FromZeros<Options> for Dst
where
    Dst: Sized + TransmuteFrom<[Zero; size_of::<Dst>()], Neglect>,
    Neglect: UnsafeTransmuteOptions
{
    fn zeroed() -> Self
    where
        Neglect: SafeTransmuteOptions
    {
        [Zero; size_of::<Self>].transmute_into()
    }

    unsafe fn unsafe_zeroed() -> Self
    where
        Neglect: UnsafeTransmuteOptions
    {
        [Zero; size_of::<Self>].unsafe_transmute_into()
    }
}
```

## Extension: Slice Casting
Transmuting the contained type of a slice is a [common operation](https://internals.rust-lang.org/t/safe-trasnsmute-for-slices-e-g-u64-u32-particularly-simd-types/2871) in cryptography. Although this RFC does not propose the addition of a concrete method for slice casting, the mechanisms proposed in this RFC make possible sound and complete slice casting abstractions; e.g.:
```rust
impl<'t, T> &'t [T] {

    /// Safely convert a `&[T]` into `&[U]`.
    pub fn transmute_into<'u, U, Neglect>(&self) -> &'u [U]
    where
        Neglect: SafeTransmuteOptions,
        &'t [T; size_of::<U>()]: TransmuteInto<&'u [U; size_of::<T>()], Neglect>
    {
        unsafe { self.unsafe_cast() }
    }

    /// Unsafely convert a `&[T]` into `&[U]`.
    pub unsafe fn unsafe_transmute_into<'u, U, Neglect>(&self) -> &'u [U]
    where
        Neglect: UnsafeTransmuteOptions,
        &'t [T; size_of::<U>()]: TransmuteInto<&'u [U; size_of::<T>()], Neglect>
    {
        let len = size_of_val(self).checked_div(size_of::<U>()).unwrap_or(0);
        unsafe {
            slice::from_raw_parts(self.as_ptr() as *const U, len)
        }
    }
}
```
The `TransmuteInto` bound statically ensures that `T` and `U` have compatible alignments, even though `.transmute_into()` isn't called in the body of this function.

## Extension: `include_data!`
[future-possibility-include_data]: #Extension-include_data

With the mechanisms proposed in this RFC, we may define a structured-data alternative to `include_bytes`:
```rust
macro_rules! include_data {
    ($file:expr) => {{
        use core::transmute::TransmuteInto;
        include_bytes!($file).transmute_into()
    }};
}
```
...and, with this, replace all instances of `unsafe` in our [earlier][motivation-including-data] motivating example:
```rust
pub fn recognize(input: &Matrix<f64, U1, U784>) -> usize
{
    static WEIGHT: &Matrix<f64, U784, U10> = include_data!("/weight.bin");

    static BIAS: &Matrix<f64, U1, U10> = include_data!("/bias.bin")

    network::recognize(input, WEIGHT, BIAS)
}
```

## Extension: Reference Casting
[ext-ref-casting]: #Extension-Reference-Casting
```rust
/// Try to convert a `&T` into `&U`.
///
/// This produces `None` if the referent isn't appropriately
/// aligned, as required by the destination type.
pub fn try_cast_ref<'t, 'u, T, U>(src: &'t T) -> Option<&'u U>
where
    &'t T: UnsafeTransmuteInto<&'u U, NeglectAlignment>,
{
    if (src as *const T as usize) % align_of::<U>() != 0 {
        None
    } else {
        // Safe because we dynamically enforce the alignment
        // requirement, whose static check we chose to neglect.
        Some(unsafe { src.unsafe_transmute_into() })
    }
}
```

## Extension: Generic Atomics
[future-possibility-generic-atomics]: #extension-generic-atomics

```rust
type LargestPlatformAtomic = u64;

pub struct Atomic<T>
where
    // ensures that size_of::<T>() <= size_of::<u64>():
    MaybeUninit<T>: TransmuteFrom<LargestPlatformAtomic>
{
    v: UnsafeCell<T>
}

impl Atomic<T>
{
    #[inline]
    pub fn load(&self, order: Ordering) -> T {
        unsafe { atomic_load(self.v.get(), order) }
    }

    #[inline]
    pub fn store(&self, val: T, order: Ordering) -> T {
        unsafe { atomic_store(self.v.get(), val, order) }
    }
}
```
