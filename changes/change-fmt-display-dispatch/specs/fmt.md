# `std::fmt` — Delta: Text Verbs as a Display/Debug Consumer

> **Delta** — replaces the "Text verbs" section, the operand columns and
> operand bullets of the "Integer verbs" and "Floating-point verbs"
> sections, the `#` row, the `#` bullet, and the precision bullet of
> "Flags, width, and precision", the runtime-failure row of "Failure
> model", and the examples of those sections; extends "Checked call
> examples" with two rows.

## Text verbs

| Verb | Operand | Output |
|------|---------|--------|
| `%v` | any | the Output rendering of the value |
| `%s` | any | identical to `%v` |
| `%q` | `string` | the value as a quoted Xulo string literal |
| `%t` | `bool` | `true` or `false` |
| `%c` | integer value | the one-scalar string of that scalar |

- `%v` and `%s` render by exactly the Output rendering the `print`
  intrinsic uses (`spec/builtins/intrinsic-functions.md` § Output): a
  user-declared type — a `struct` or an `enum` — consults its
  implementations wherever the rendering reaches it, as the operand itself
  or in any structural position; every other value is rendered by the
  table. The two verbs mean the same thing; both are spelled because
  readers arrive from Go with either.
- The consult asks under a `Format` built from the directive
  (`spec/types/traits.md`, Built-in formatting traits): `precision` is the
  directive's precision, or `null` when none was written; `alternate` is
  true when `#` was written. A `Display` implementation is asked first and
  its return is the verb's text; a `Debug` implementation is asked when no
  `Display` exists, and its return is the text in the same way. The text is
  used as returned: precision, having been handed over, is never re-applied
  as a cut, and width, alignment, and padding are laid out by `format` over
  the returned text as over any other.
- When no implementation is consulted — a base type, `null`, a container,
  an `enum` whose type implements neither — the value is rendered by the
  table; a written precision then cuts that rendered text to that many
  scalars.
- A `struct` whose type implements neither `Display` nor `Debug` is a
  runtime failure — the argument-domain counterpart of the `E0212` the same
  value's interpolation reports.
- `%q` wraps the value in double quotes and escapes exactly what a Xulo
  string literal escapes (`spec/grammar.md`, `EscapeSeq`): `"` as `\"`, `\`
  as `\\`, and U+000A, U+000D, and U+0009 as `\n`, `\r`, `\t`; any other
  scalar below U+0020, and U+007F itself, takes its four-digit `\uXXXX`
  escape. The result is a valid literal for the same value.
- `%c` builds the one-scalar string of its operand — the scalar-to-text
  step the prelude has no member for. An integer operand — an `int` or a
  fixed-bit integer, the Integer verbs' operand — outside `0 …= 0x10FFFF`,
  or in the surrogate range U+D800 … U+DFFF, is a runtime failure.

```xulo
print(std::fmt::format("%q", ["hi"]))           // runs: "hi"
print(std::fmt::format("%q", ["a\n"]))          // runs: "a\n"
std::fmt::format("%c", [8364])                  // ok: "€"
std::fmt::format("%t / %v", [true, null])       // ok: "true / null"
std::fmt::format("%v", [[1, [2, 3]]])           // ok: "[1, [2, 3]]"
```

The consult, in full:

```xulo
#Derive(Debug)
struct Point { x: int, y: int }

struct Money { cents: int }

impl Display for Money {
  fn display(self, format: Format): string {
    std::math::toFixed(self.cents / 100.0, format.precision ?? 2)
  }
}

std::fmt::format("%v", [Point(x: 1, y: 2)])     // ok: "Point(x: 1, y: 2)" — no Display; Debug asked
std::fmt::format("%s", [Money(cents: 12345)])   // ok: "123.45" — the default precision
std::fmt::format("%.1s", [Money(cents: 12345)]) // ok: "123.5" — the precision is the Format's
std::fmt::format("%#s", [Money(cents: 12345)])  // ok: "123.45" — alternate: true; this display reads only precision

struct Bare { x: int }
std::fmt::format("%v", [Bare(x: 1)])            // runs: runtime failure — neither Display nor Debug
```

## Integer verbs

| Verb | Operand | Output |
|------|---------|--------|
| `%d` | integer value | decimal |
| `%x` `%X` | integer value | hexadecimal, lower- and upper-case digits |
| `%o` | integer value | octal |
| `%b` | integer value | binary |

- An **integer value** is an `int` or a value of a fixed-bit integer type —
  `i8` … `i64`, `u8` … `u64`, `usize`, `isize` — and every verb renders
  the value in its own base whatever the operand's type was.
- The sign is preserved and the digits are the absolute value's:
  `format("%x", [-255])` is `-ff`. The one `int` value with no positive
  counterpart, `-9223372036854775808`, is digitized from its
  two's-complement image — `-8000000000000000` under `%x` — so no operand
  can overflow the magnitude; a fixed-bit operand follows the same rule at
  its own type's minimum.
- `#` is the alternate form: `%#x`, `%#X`, `%#o`, and `%#b` put `0x`, `0X`,
  `0o`, and `0b` after the sign.
- A `float` operand — `float`, `f32`, `f64`, a whole one included — is a
  runtime failure: the integer verbs take an integer value, whatever the
  argument's static type was.

```xulo
print(std::fmt::format("%d", [-42]))            // runs: -42
print(std::fmt::format("%x / %#X", [255, 255])) // runs: ff / 0XFF
print(std::fmt::format("%b", [5]))              // runs: 101
let small: u8 = 255
std::fmt::format("%x", [small])                 // ok: "ff"
```

## Floating-point verbs

| Verb | Operand | Output |
|------|---------|--------|
| `%f` | `number`, `f32`, `f64` | fixed point — `precision` digits after the point (default 6) |
| `%e` `%E` | `number`, `f32`, `f64` | scientific — one digit, `.`, `precision` digits, then `e` or `E` and a signed exponent of at least two digits (default 6) |
| `%g` `%G` | `number`, `f32`, `f64` | the shorter of the `%f` and `%e` forms, `%G` spelling the exponent with `E` |

- Rounding is the `std::math::toFixed` rule — `precision` places, halfway
  away from zero — and its overflow note applies to `%f` unchanged
  ([`math.md`](math.md)). An integer operand is converted to the `float`
  nearest its value — exact within ±2⁵³, ties to even beyond — and an `f32`
  operand is widened to the `float` it exactly names; the verb then applies
  to that value.
- A negative zero keeps its sign (`%f` of `-0.0` is `-0.000000`, `%e`
  `-0.000000e+00`), and the special values ignore precision to render as
  `NaN`, `Infinity`, and `-Infinity` — the spellings `str` and `std::math`
  already fix. The flags `+`, ` `, `#`, and `0` change nothing about those
  three; `width` and `-` still place them.
- `%g` reads `precision` as **significant digits** (`1 …= 17`): it rounds to
  that many and then drops trailing zeros from the `%f` form — and the point,
  when nothing follows it. The form is the `%e` one when the decimal
  exponent of the leading digit is below `−4` or reaches `precision`, and
  the `%f` one otherwise. Omitted, `%g` prints the shortest digit string
  that reads back as the same value: for an `f32` operand the digits of the
  `f32` row of the Output table — the same digits `%v` prints — and
  otherwise the digits of the `float` row; the form rule then takes
  `precision` to be that string's digit count, so `0.1` prints `0.1` and
  `1e20` prints `1e+20`.
- `%f` and `%e` give six digits after the point when precision is omitted,
  the Go and C default.

```xulo
print(std::fmt::format("%f", [2.5]))           // runs: 2.500000
print(std::fmt::format("%.2f", [3.14159]))     // runs: 3.14
print(std::fmt::format("%e", [1234.5]))        // runs: 1.234500e+03
print(std::fmt::format("%.3g", [0.1]))         // runs: 0.1
print(std::fmt::format("%g", [1e-7]))          // runs: 1e-07
print(std::fmt::format("%g", [1e20]))          // runs: 1e+20 — one significant digit; 20 reaches it
let half: f32 = 0.5
std::fmt::format("%.1f", [half])               // ok: "0.5" — widened to its exact float
let one: f32 = 1
let three: f32 = 3
std::fmt::format("%v", [one / three])          // ok: "0.33333334" — the f32 row of the Output table
std::fmt::format("%g", [one / three])          // ok: "0.33333334" — the same digits, not the widened float's
```

## Flags, width, and precision

| Decoration | Meaning | Verbs |
|------------|---------|-------|
| `-` | pad to the right instead of the left | every verb |
| `0` | pad with `0` instead of spaces | `%d` `%x` `%X` `%o` `%b` `%f` `%e` `%E` `%g` `%G` |
| `+` | a positive value carries `+` | the numeric verbs (the same set) |
| ` ` | a positive value's sign position is a space | the numeric verbs (the same set) |
| `#` | the alternate form — for `%v` `%s`, the `Format` a user implementation is asked under carries `alternate: true` | `%x` `%X` `%o` `%b` `%v` `%s` |

- **Width** is a minimum, in Unicode scalar values, for every verb but
  `%%`: a shorter result is padded and a longer one passes through. Its
  digits MUST fit an `int`; a longer run is a runtime failure. Padding
  with `0` is sign-aware — sign, then the `#` prefix, then the zeros, then
  the digits — so `%#08x` of `255` is `0x0000ff` and `%08.2f` of `-3.1` is
  `-0003.10`.
- **Precision** is meaningful for `%f`, `%e`, `%E`, `%g`, `%G` (the digits
  it names); for `%v` and `%s` (handed to a user implementation as the
  `Format`'s precision, and otherwise cutting the table-rendered text to
  that many scalars); and for `%q` (the text is cut to that many scalars).
  On any other verb a precision is a runtime failure. Its digits MUST fit
  an `int` — a longer run is a runtime failure — and for the text verbs
  the domain is every `int` from `0` up: `0` is a request, not an absence,
  and a cut longer than the text leaves it unchanged.
- Two combinations carry the C rules: `0` is ignored when `-` is given, and
  `+` outranks ` `. Every other combination is ordinary.
- `#` on `%v` and `%s` is well-formed for every operand: a value with no
  implementation asked renders by the table, and the flag has no effect on
  it.
- The decoration table is closed: a flag outside its verb's column is a
  runtime failure — `%+s` and `%#d` are mistakes, not no-ops.

```xulo
print(std::fmt::format("%-6s|", ["hi"]))       // runs: hi    |
print(std::fmt::format("%+8.2f", [3.1]))       // runs:    +3.10
print(std::fmt::format("%.3s", ["hello"]))     // runs: hel
print(std::fmt::format("%.3s", [[1, 2, 3]]))   // runs: [1,   — a table rendering is cut
```

## Failure model

| Kind | Members |
|------|---------|
| Total | none — `format` is defined only for a well-formed template and a matching argument list |
| Runtime failure | `format` — every malformed template (a verb outside the table, a repeated flag, a flag the verb does not accept, a decorated `%%`, a `%` with no verb, a `.` with no precision digits, a width or precision whose digits do not fit an `int`) and every mismatched argument list (a directive count disagreeing with `args`, an operand of the wrong kind for its verb — a `float` for `%d`, a `struct` whose type implements neither `Display` nor `Debug` under `%v` or `%s` — a precision outside its verb's domain — `0 …= 15` for `%f`/`%e`/`%E`, `1 …= 17` for `%g` — or a `%c` scalar outside `0 …= 0x10FFFF` or in U+D800 … U+DFFF) |
| Fallible (`Result`) | none |

A template is the program's own text and the arguments are the program's own
values, so every failure here is an argument domain — the class § 5.3 of
[`README.md`](README.md) leaves as a runtime failure, the shape
`std::math::toFixed`'s `digits` already takes. The module never meets the
outside world, so there is no failure for § 5.1's `Result` to carry.

## Checked call examples

```xulo
std::fmt::format("%6.2f", [3.14159])            // ok: "  3.14"
std::fmt::format("100%%")                       // ok: "100%"
std::fmt::format("%-8s|%d", ["hi", 42])         // ok: "hi      |42"
std::fmt::format("%#08x", [255])                // ok: "0x0000ff"
std::fmt::format("%.3g, %q", [0.1, "hi"])       // ok: "0.1, \"hi\""
std::fmt::format("%v", [[1, [2, 3]]])           // ok: "[1, [2, 3]]"
std::fmt::format("%.3s", [[1, 2, 3]])           // ok: "[1,"
std::fmt::format("%d")                          // runs: runtime failure — one directive and no argument
std::fmt::format("%d", ["a"])                   // runs: runtime failure — `%d` takes an integer value
std::fmt::format("%z", [1])                     // runs: runtime failure — no verb `z`
std::fmt::format("%d", [1], [2])                // error[E0207]: expected 1 to 2 arguments, found 3
std::fmt::format(42, [1])                       // error[E0202]: expected `string`, found `int`
std::fmt::format<int>("%d", [1])                // error[E0807]: `std::fmt::format` accepts no type arguments
Fmt::format("%d", [1])                          // error[E0607]: `Fmt` lives under `std`: use `std::fmt::format`
```
