- Feature Name: generic_atomic
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

[safer transmutation]: 0000-safe-transmute.md

# Summary
[summary]: #summary

This is a **library extension** to [safer transmutation] (i.e., it does not require additional compiler support) which introduces `Atomic<T>`.

# Motivation
[motivation]: #motivation

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

The [safer transmutation] mechanisms permit a truly-generic `Atomic<T>` type.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Using the [safe transmutation] mechanisms, `Atomic<T>` might be defined like so:

```rust
type LargestPlatformAtomic = u64; // platform-dependent

pub struct Atomic<T>
where
    T: SizeLtEq<LargestPlatformAtomic>
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