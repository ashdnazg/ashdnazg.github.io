---
layout: post
title:  "16-bit Fast Inverse Square Root"
date:   2025-08-23
visible: 1
---

If you're not familiar with the [Fast Inverse Square Root](https://en.wikipedia.org/wiki/Fast_inverse_square_root) algorithm, I thoroughly recommend reading about it before continuing. Not necessarily because the details are important, but because it's such a mind-blowingly impressive hack.

If you are already familiar with it, you might remember that it's based on a magical constant (0x5F3759DF[^1]), carefully chosen to minimise the algorithm's relative error.

This constant was chosen by smart people who reasoned about how the algorithm works and derived formulas for picking the best constant for the job. Using the same process, they also derived the constant for double precision (64-bit) floating point numbers - 0x5FE6EB50C7B537A9. If you're into that kind of thing, you can read all about it in Matthew Robertson's [A Brief History of InvSqrt](https://mrober.io/papers/rsqrt.pdf)

Nowadays, with 16-bit floating point numbers being all the rage, whether [half-precision](https://en.wikipedia.org/wiki/Half-precision_floating-point_format) or [bfloat16](https://en.wikipedia.org/wiki/Bfloat16_floating-point_format), I wondered if anybody derived the magic number for these formats. A short search produced no results, so I figured I might just do it myself.

Unfortunately, this late at night I'm incapable of following all the sophisticated math done by the aforementioned smart people, but I do have one advantage - there are only 2<sup>16</sup> 16-bit floating point numbers[^2] and only 2<sup>16</sup> possible magic numbers, totalling just over 4 billion combinations. So let's take an inspiration from one of my favourite articles - [There are Only Four Billion Floats–So Test Them All!](https://randomascii.wordpress.com/2014/01/27/theres-only-four-billion-floatsso-test-them-all) - and simply test them all!

With the [half](https://crates.io/crates/half) library giving me a pleasant interface in Rust for 16-bit floats, and the algorithm already existing, all that is left to do is to hack together two nested loops:

```rust
use half;
use num_traits::float::Float as _;

// Switch which line is commented out to choose between half and bfloat16
type f16 = half::f16;
// type f16 = half::bf16;

fn isqrt_half(x: f16, magic: u16) -> f16 {
    const THREE_HALVES: f16 = f16::from_f32_const(1.5);

    let x2 = x * f16::from_f32_const(0.5);
    let mut y = x;
    let mut i = y.to_bits();
    i = magic.wrapping_sub(i >> 1);
    y = f16::from_bits(i);
    y = y * (THREE_HALVES - x2 * y * y);

    y
}

fn main() {
    let mut max_err = [f16::ZERO; u16::MAX as usize + 1];
    for magic in u16::MIN..=u16::MAX {
        for i in u16::MIN..=u16::MAX {
            let f = f16::from_bits(i);
            if !f.is_normal() || f <= f16::ZERO {
                continue;
            }
            let expected = f16::from_f64(f.to_f64().sqrt().recip());
            let actual = isqrt_half(f, magic);
            let err = (actual - expected).abs() / expected;
            max_err[magic as usize] = err.max(max_err[magic as usize]);
        }
    }
    let (constant, error) = max_err
        .iter()
        .enumerate()
        .filter(|(_, c)| c.is_normal())
        .min_by(|(_, a), (_, b)| a.partial_cmp(b).unwrap())
        .unwrap();
    println!("Best magic: {constant:#04X}, Relative error: {}", error);
}
```

Less than a minute later, we get the results: for the half-precision format, the best constant is __0x59B7__ and for the bfloat16 format, the constant is __0x5F35__[^3].

Let's compare the new constants with the ones for 32- and 64-bit floats:

| Format          | Constant           | Maximal Relative Error        |
|-----------------|--------------------|-------------------------------|
| half            | 0x59B7             | 0.0028362274                  |
| bfloat16        | 0x5F35             | 0.008483887                   |
| single (32-bit) | 0x5F375A86         | 0.0017512378                  |
| double (64-bit) | 0x5FE6EB50C7B537A9 | 0.0017511837                  |

We see that the accuracy of the algorithm using the half format is much better than using bfloat16, likely due to the latter's lower precision[^4]. But even with bfloat16, the maximal error is lower than 1%, a testament to the robustness of the algorithm.

Is this useful for anything? Probably not, since the algorithm itself is more or less [obsolete](https://en.wikipedia.org/wiki/Fast_inverse_square_root#Obsolescence). Nevertheless, these magical constants deserve to be put on the internet, so here they are.

---

[^1]: Later the constant 0x5F375A86 was shown to produce better results.

[^2]: We actually have even less than 2<sup>16</sup>, since the algorithm is only defined for normal positive numbers, of which there are 30720 half precision floats and 32512 bfloat16s.

[^3]: When skipping the Newton-Raphson step, the constant for half is 0x59BB and the constant for bfloat16 is 0x5F37.

[^4]: bfloat16 has 7 bits of mantissa, while half has 10.
