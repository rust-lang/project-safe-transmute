- Feature Name: include_data
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

[safer transmutation]: 0000-safe-transmute.md

# Summary
[summary]: #summary

This is a **library extension** to [safer transmutation] (i.e., it does not require additional compiler support) which introduces `include_data!`, a safe, generic alternative to `include_bytes!`.

# Motivation
[motivation]: #motivation

Rust's [`include_bytes!` macro](https://doc.rust-lang.org/core/macro.include_bytes.html) lets you statically include the contents of a file into your executable's binary. The builtin is a quick-and-dirty solution for packaging data with your executable, and perhaps even helping the compiler optimize your code. **Unfortunately, it's difficult to use correctly.** Consider:
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
This is memory-unsafe, because `RAW_WEIGHT` and `RAW_BIAS` might not meet the alignment requirements of `Matrix<f64,_,_>`. This fix is subtle:
```rust
pub fn recognize(input: &Matrix<f64, U1, U784>) -> usize
{
    static RAW_WEIGHT : &'static [u8; 62_720] = include_bytes!("/weight.bin");

    static RAW_BIAS : &'static [u8; 80] = include_bytes!("/bias.bin");

    let WEIGHT: &Matrix<f64, U784, U10> = unsafe{ &mem::transmute(*RAW_WEIGHT) };

    let BIAS: &Matrix<f64, U1, U10> = unsafe{ &mem::transmute(*RAW_BIAS) };

    network::recognize(input, WEIGHT, BIAS)
}
```
Even still, there is potential for undefined behavior: `mem::transmute` does almost nothing to ensure that the included bytes are a *valid* instance of `Matrix<f64,_,_>`. This RFC proposes a **safe** `include_transmute!` macro that lacks these disadvantages; e.g.:
```rust
pub fn recognize(input: &Matrix<f64, U1, U784>) -> usize
{
    static WEIGHT: &Matrix<f64, U784, U10> = include_transmute!("/weight.bin");

    static BIAS: &Matrix<f64, U1, U10> = include_transmute!("/weight.bin");

    network::recognize(input, &WEIGHT, &BIAS)
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Using the [safe transmutation] mechanisms, `include_transmute!` might be defined like so:

```rust
macro_rules! include_transmute {
    ($file:expr) => {{
        use core::convert::transmute::*;
        &safe_transmute<_, _, ()>(*include_bytes!($file))
    }};
}
```