# `std::math`

`std::math` holds the mathematical constants and functions: ordinary values
and operations over `float` and `int`, with no `Result`, no options, and
no state — except the two members that draw from the module's random stream.
Every failure
this chapter states is an argument domain, pinned in § Failure model.

The module is host-independent: the same inputs give the same results on every
machine, up to the last bits of transcendental functions (§ 5.7 of
[`README.md`](README.md)).

This module is the former prelude `Math` namespace moved under `std`; the
migration is tracked in [`../README.md`](../README.md) § 1.3. The qualified
path needs no import:

```xulo
std::math::floor(3.7)                  // ok: 3
std::math::PI                          // ok: an ordinary float value
print(std::math::toFixed(3.14159, 2))  // ok: 3.14
```

Import forms are the six of [`README.md`](README.md) § 2.2:

```xulo
use std::math::*;               // the constants and every function
use std::math as m;
m::sqrt(2.25)                   // ok: 1.5
use std::math::{floor, toFixed};
```

## Constants

| Member | Type | Value |
|--------|------|-------|
| `std::math::PI` | `float` | 3.141592653589793 |
| `std::math::E` | `float` | 2.718281828459045 |
| `std::math::TAU` | `float` | 6.283185307179586 |
| `std::math::INFINITY` | `float` | positive infinity |
| `std::math::NAN` | `float` | not a number |

The constants are ordinary `float` values: they read, compare, and print like
any other. `str` and interpolation render them as `NaN` and `Infinity`
(`-Infinity` when negative), and a negative zero prints as `0`.

```xulo
let halfPi = std::math::PI / 2        // ok
let big = std::math::INFINITY         // ok: Infinity
let nope = std::math::NAN             // ok: NaN — and nope != nope is true
```

## Basic functions

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::math::abs` | `fn(x: number): number` | absolute value |
| `std::math::min` | `fn(a: number, b: number): number` | the smaller of two values |
| `std::math::max` | `fn(a: number, b: number): number` | the larger of two values |
| `std::math::sign` | `fn(x: number): number` | `-1`, `0`, or `1` by the sign of `x` |
| `std::math::sqrt` | `fn(x: float): float` | square root |
| `std::math::cbrt` | `fn(x: float): float` | cube root |
| `std::math::pow` | `fn(base: float, exp: float): float` | `base` raised to `exp` |
| `std::math::hypot` | `fn(a: float, b: float): float` | `sqrt(a² + b²)`, computed without intermediate overflow |
| `std::math::clamp` | `fn(x: number, lo: number, hi: number): number` | `x` limited to `lo … hi` |

- `abs`, `min`, `max`, `sign`, and `clamp` are declared over `number`: an
  `int` or `float` operand is accepted by union subtyping
  (`number = int | float`), a fixed-bit operand is not, and neither is a
  `string`.
- **`abs` of the most negative `int`** — `-9223372036854775808`, the one
  value with no positive counterpart — yields that value itself: the
  two's-complement image is all the `int` can hold (the rule Java gives
  `Math.abs` for `long`). `abs` stays total.
- A `NaN` operand propagates through `min` and `max`: the result is `NaN`,
  matching `abs(NaN)` and every other member below.
- `sign` answers with the plain values `-1`, `0`, `1` — of the operand's
  kind: an `int` operand gives an `int` value, a `float` operand a `float`
  value. Either zero gives `0`, `-0` included; `NaN` gives `NaN`.
- `sqrt` of a negative value yields `NaN`; `cbrt` is defined for every
  `float`, negative values included.
- `hypot` is symmetric in its operands and never overflows on its own
  squaring: `hypot(1e308, 1e308)` is finite where the naive
  `sqrt(a*a + b*b)` is `Infinity`, and `hypot(0, 0)` is `0.0`. An infinite
  operand gives infinity even when the other is `NaN` — IEEE-754's
  ordering of the special cases — and otherwise a `NaN` operand gives
  `NaN`.
- `clamp` passes `x` through when it lies within `lo … hi` and returns the
  bound it fell past when it does not. **`lo` greater than `hi` is a
  runtime failure** (`spec/memory-and-runtime.md`) — an empty range admits
  no value — and a `NaN` operand anywhere gives `NaN`.

```xulo
print(std::math::hypot(3, 4))          // runs: 5
print(std::math::sign(-2.5))           // runs: -1
print(std::math::clamp(9, 0, 5))       // runs: 5
std::math::clamp(1, 5, 0)              // runtime failure: lo greater than hi
```

## Rounding and narrowing

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::math::floor` | `fn(x: float): int` | largest integer not above `x` |
| `std::math::ceil` | `fn(x: float): int` | smallest integer not below `x` |
| `std::math::round` | `fn(x: float): int` | nearest integer to `x` |
| `std::math::trunc` | `fn(x: float): int` | `x` with its fractional part discarded |

- `floor`, `ceil`, and `trunc` discard the fraction toward negative infinity,
  toward positive infinity, and toward zero; they return `int`. These four
  functions are the language's narrowing operations — a `float` value becomes
  an `int` through one of them and through nothing else
  (`spec/type-system/coercion.md`).
- `round` returns the nearest `int`, and a value exactly halfway between two
  integers rounds **away from zero**: `round(2.5)` is `3` and `round(-2.5)`
  is `-3`.
- **Range.** The result of these four functions is an `int`; an integral
  result outside `int`'s range (`-2⁶³ … 2⁶³−1`) is a runtime failure — the
  same class as an overflowing `+` on `int`
  (`spec/memory-and-runtime.md`).

```xulo
let whole = std::math::floor(3.7)      // ok: 3, of type int
let n: int = std::math::round(2.5)     // ok: 3
print(std::math::floor(-3.2))          // runs: -4
print(std::math::trunc(-3.2))          // runs: -3
let huge = std::math::floor(1e308)     // runtime failure: outside int's range
```

## Integer and bit functions

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::math::wrappingAdd` | `fn(a: int, b: int): int` | `a + b` modulo 2⁶⁴, as two's complement |
| `std::math::wrappingSub` | `fn(a: int, b: int): int` | `a − b` modulo 2⁶⁴, as two's complement |
| `std::math::wrappingMul` | `fn(a: int, b: int): int` | `a × b` modulo 2⁶⁴, as two's complement |
| `std::math::popCount` | `fn(x: int): int` | the number of 1 bits in `x` |
| `std::math::countLeadingZeros` | `fn(x: int): int` | leading 0 bits — 64 when `x` is 0 |
| `std::math::countTrailingZeros` | `fn(x: int): int` | trailing 0 bits — 64 when `x` is 0 |
| `std::math::rotateLeft` | `fn(x: int, amount: u64): int` | `x` rotated left by `amount` modulo 64 |
| `std::math::rotateRight` | `fn(x: int, amount: u64): int` | `x` rotated right by `amount` modulo 64 |

- The `wrapping*` functions never trap: their result is the unique `int`
  congruent to the given sum, difference, or product modulo 2⁶⁴, taken as
  two's complement, and the operator overflow rule does not apply to them
  (`spec/types/primitive-types.md`).
- `popCount`, `countLeadingZeros`, and `countTrailingZeros` count bits of the
  64-bit two's-complement representation of their argument.
- The rotations take that same representation and interpret `amount`
  **modulo 64**, so `0` and `64` are the identity and every `u64` is in
  domain — no amount is rejected.

```xulo
let maxInt = 9223372036854775807
print(std::math::wrappingAdd(maxInt, 1))   // runs: -9223372036854775808
print(std::math::countLeadingZeros(0))     // runs: 64
print(std::math::rotateLeft(1, 64))        // runs: 1
```

## Formatting

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::math::toFixed` | `fn(x: float, digits: int): string` | the decimal numeral of `x` with exactly `digits` digits after the decimal point |

- The result has exactly `digits` fractional digits: trailing zeros are
  kept — a fixed width is the point of the function — and the value is
  rounded at the `digits`-th place with the same halfway-away-from-zero rule
  as `round`.
- The sign is rendered explicitly, so a negative operand carries `-`, and a
  negative zero renders as `-0` (or `-0.00` at `digits = 2`).
- The special values ignore `digits` and render as `NaN`, `Infinity`, and
  `-Infinity`.
- **`digits` must be in `0 …= 15`; any other value is a runtime failure**
  (`spec/memory-and-runtime.md`).
- If scaling the operand by `10^digits` overflows to infinity, the operand is
  formatted directly at the requested width instead of being rounded.

```xulo
print(std::math::toFixed(3.14159, 2))   // runs: 3.14
print(std::math::toFixed(2.5, 0))       // runs: 3
print(std::math::toFixed(-2.5, 0))      // runs: -3
print(std::math::toFixed(3.1, 2))       // runs: 3.10
print(std::math::toFixed(std::math::NAN, 4))   // runs: NaN
let bad = std::math::toFixed(1.5, 16)   // runtime failure: digits outside 0 …= 15
```

## Trigonometric functions

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::math::sin` | `fn(x: float): float` | sine of an angle in radians |
| `std::math::cos` | `fn(x: float): float` | cosine of an angle in radians |
| `std::math::tan` | `fn(x: float): float` | tangent of an angle in radians |
| `std::math::asin` | `fn(x: float): float` | arcsine, in radians |
| `std::math::acos` | `fn(x: float): float` | arccosine, in radians |
| `std::math::atan` | `fn(x: float): float` | arctangent, in radians |
| `std::math::atan2` | `fn(y: float, x: float): float` | arctangent of `y / x`, quadrant-aware |

- Angles are in radians; `asin` and `acos` yield `NaN` outside `-1.0 … 1.0`.

## Logarithmic and exponential functions

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::math::log` | `fn(x: float): float` | natural logarithm |
| `std::math::log2` | `fn(x: float): float` | base-2 logarithm |
| `std::math::log10` | `fn(x: float): float` | base-10 logarithm |
| `std::math::log1p` | `fn(x: float): float` | natural logarithm of `1 + x`, accurate for small `x` |
| `std::math::exp` | `fn(x: float): float` | `e` raised to `x` |
| `std::math::expm1` | `fn(x: float): float` | `eˣ − 1`, accurate for small `x` |

- `log`, `log2`, and `log10` yield `-Infinity` at `0` and `NaN` for a
  negative argument; `exp` yields `Infinity` once its result overflows.
- `log1p` and `expm1` exist for precision: written the naive way,
  `log(1 + x)` loses `x` entirely when it is far below the last bit of `1`
  (`log(1 + 1e-17)` rounds to `0`) and `exp(x) − 1` loses it the same way;
  this pair returns the full-precision result, and they are inverses of
  each other. Their domains follow the pair they repair: `log1p(-1)` is
  `-Infinity`, `log1p` below `-1` is `NaN`, and `expm1` overflows to
  `Infinity`.

```xulo
print(std::math::log1p(1e-17))   // runs: 1e-17 — where log(1 + 1e-17) rounds to 0
print(std::math::expm1(1e-17))   // runs: 1e-17 — where exp(1e-17) - 1 rounds to 0
```

### IEEE-754 behaviour (whole module)

Every `float` parameter and result follows IEEE-754 binary64: an operation
outside its domain yields `NaN`, an overflow yields the appropriate infinity,
and the special values propagate through further operations. No `std::math`
member returns `Result`, and no *float* operation fails or panics — `NaN` and
the infinities *are* its failure handling (`spec/types/primitive-types.md`);
the only failures are the argument domains § Failure model lists.

## Random

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::math::random` | `fn(): float` | the next pseudo-random value in `[0.0, 1.0)` |
| `std::math::randomInt` | `fn(lo: int, hi: int): int` | the next pseudo-random integer in `lo … hi`, both ends included |

- The stream starts at a seed drawn from system entropy when the process
  starts; successive calls yield successive values of one generator, each
  53 random bits scaled by 2⁻⁵³ — `0.0` is possible, `1.0` is not.
- The sequence is **not cryptographic** and not reproducible across runs; do
  not use it for security, salts, or anything that needs a predictable seed.
- `randomInt` draws from that **same stream**: interleaving it with
  `random` advances one sequence, under the same entropy-drawn seed and the
  same non-cryptographic warning.
- Every integer in `lo … hi` is **equally likely** — uniform over the
  range, from a full 64-bit draw, so a range of any size up to the entire
  `int` span draws uniformly (the 53-bit scale of `random` does not apply
  to it). Both ends are included: `randomInt(1, 6)` is a die.
- **`lo` greater than `hi` is a runtime failure**
  (`spec/memory-and-runtime.md`) — an empty range has no member to draw.

```xulo
let r = std::math::random()
let inRange = r >= 0.0 && r < 1.0    // ok: always true
let roll = std::math::randomInt(1, 6)   // ok: an int in 1 …= 6
std::math::randomInt(6, 1)              // runtime failure: lo greater than hi
```

## Failure model

| Kind | Members |
|------|---------|
| Total | every member — constants, `abs`/`min`/`max`, `sqrt`/`cbrt`/`pow`/`hypot`/`sign`, the four rounding functions, the eight integer/bit functions, the trigonometric and logarithmic groups, `random` |
| Runtime failure | `toFixed` with `digits` outside `0 …= 15`; a rounding function whose integral result falls outside `int`'s range; `clamp` and `randomInt` with `lo` greater than `hi` |
| Fallible (`Result`) | none |

## Checked call examples

```xulo
std::math::min(3, 9)                  // ok
std::math::sqrt(4)                    // ok: 2 — an integer literal coerces to float
std::math::hypot(3, 4)                // ok: 5
std::math::sign(-2.5)                 // ok: -1
std::math::clamp(9, 0, 5)             // ok: 5
std::math::min("a", "b")              // error[E0202]: expected `number`, found `string`
std::math::abs(1, 2)                  // error[E0207]: expected 1 argument, found 2
std::math::PI(2.0)                    // error[E0204]: not callable
std::math::PI                         // ok on its own: an ordinary float value
Math::PI                              // error[E0607]: `Math` lives under `std`
std::math::arcsin(0.5)                // error[E0601]: no member `arcsin`
```
