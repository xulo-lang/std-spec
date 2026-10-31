# `std::fmt`

`std::fmt` holds the one member that assembles formatted text: a template of
literal text and `%` directives, values supplied in order, and the text each
directive asks for. The directives carry the layout vocabulary Go's
`fmt.Sprintf` and Rust's `format!` share — verb, flags, width, and precision
— so numbers, text, and booleans are laid out in place rather than built
piece by piece and concatenated.

The module is small and closed: `format` is its whole face. It touches
nothing but its own arguments — no I/O, no host, no state — so it is
portable and deterministic: the same template and the same values give the
same text on every machine (§ 5.5 and § 5.7 of [`README.md`](README.md)).
No member returns `Result`. A malformed template or a mismatched argument is
an argument domain — a mistake in the program's own text, the class § 5.3
lets stop the program — never a failure of the outside world.

The qualified path needs no import:

```xulo
std::fmt::format("%6.2f", [3.14159])           // ok: "  3.14"
print(std::fmt::format("%-8s|", ["hi"]))       // runs: hi      |
```

Import forms are the six of [`README.md`](README.md) § 2.2:

```xulo
use std::fmt::*;                       // binds format
use std::fmt::format;
format("%d", [42])                     // ok: "42"
use std::fmt as f;
f::format("%x", [255])                 // ok: "ff"
```

## The member

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fmt::format` | `fn(template: string, args: List<any> = []): string` | the template's literal text, with each directive replaced by its formatted argument |

- `args` supplies the values left to right: the first directive takes the
  first element, the second the next, and so on. It may be omitted when the
  template names no arguments — `format("100%%")` is `"100%"` — so a call
  takes one or two arguments ([`README.md`](README.md) § 4).
- The list is checked as `List<any>`: the literal `[1, "a"]` at the call
  site is the ordinary spelling, and the checker looks at no element
  ([`../README.md`](../README.md) § 1.3, item 2). Whether a value suits its
  verb is decided at run time, on the value itself.
- The result is new text; `format` modifies neither the template nor the
  values.

## Directives

A template is literal text and directives. Everything outside a directive
copies through unchanged; a directive has this shape:

```text
Directive = "%" [ Flag { Flag } ] [ Width ] [ "." Precision ] Verb ;
Flag      = "-" | "+" | "0" | " " | "#" ;
Width     = Digit { Digit } ;
Precision = Digit { Digit } ;
Verb      = "v" | "s" | "q" | "t" | "c"
          | "d" | "x" | "X" | "o" | "b"
          | "f" | "e" | "E" | "g" | "G" ;
```

- `%%` is the escape for a literal `%`. It consumes no argument and takes no
  flags, width, or precision.
- Every other directive consumes exactly one argument, in order; the number
  of argument-taking directives and the size of `args` MUST agree.
- Width and precision are written as digits in the template. Go's `*`, which
  draws them from the argument list, and its explicit argument indexes
  (`%[2]d`) do not exist here.
- Each flag appears at most once and in any order. A template that fits
  nothing above is a runtime failure; the Failure model enumerates the
  cases.

## Text verbs

| Verb | Operand | Output |
|------|---------|--------|
| `%v` | any | the `str` rendering of the value |
| `%s` | any | identical to `%v` |
| `%q` | `string` | the value as a quoted Xulo string literal |
| `%t` | `bool` | `true` or `false` |
| `%c` | `int` | the one-scalar string of that scalar |

- `%v` and `%s` render by exactly the table the `str` intrinsic uses
  (`spec/builtins/intrinsic-functions.md`): a `string` prints its characters
  with no quotes and a container prints its elements, so `format("%s",
  ["hi"])` is `hi` and `format("%v", [42])` is `42`. The two verbs mean the
  same thing; both are spelled because readers arrive from Go with either.
- `%q` wraps the value in double quotes and escapes exactly what a Xulo
  string literal escapes (`spec/grammar.md`, `EscapeSeq`): `"` as `\"`, `\`
  as `\\`, and U+000A, U+000D, and U+0009 as `\n`, `\r`, `\t`; any other
  scalar below U+0020, and U+007F itself, takes its four-digit `\uXXXX`
  escape. The result is a valid literal for the same value.
- `%c` builds the one-scalar string of its operand — the scalar-to-text step
  the prelude has no member for. A scalar outside `0 …= 0x10FFFF`, or in the
  surrogate range U+D800 … U+DFFF, is a runtime failure.

```xulo
print(std::fmt::format("%q", ["hi"]))          // runs: "hi"
print(std::fmt::format("%q", ["a\n"]))         // runs: "a\n"
std::fmt::format("%c", [8364])                 // ok: "€"
std::fmt::format("%t / %v", [true, null])      // ok: "true / null"
```

## Integer verbs

| Verb | Operand | Output |
|------|---------|--------|
| `%d` | `int` | decimal |
| `%x` `%X` | `int` | hexadecimal, lower- and upper-case digits |
| `%o` | `int` | octal |
| `%b` | `int` | binary |

- The sign is preserved and the digits are the absolute value's:
  `format("%x", [-255])` is `-ff`. The one value with no positive
  counterpart, `-9223372036854775808`, is digitized from its two's-complement
  image — `-8000000000000000` under `%x` — so no operand can overflow the
  magnitude.
- `#` is the alternate form: `%#x`, `%#X`, `%#o`, and `%#b` put `0x`, `0X`,
  `0o`, and `0b` after the sign.
- A `float` operand is a runtime failure, a whole one included: the integer
  verbs take an `int` value, whatever the argument's static type was.

```xulo
print(std::fmt::format("%d", [-42]))           // runs: -42
print(std::fmt::format("%x / %#X", [255, 255])) // runs: ff / 0XFF
print(std::fmt::format("%b", [5]))             // runs: 101
```

## Floating-point verbs

| Verb | Operand | Output |
|------|---------|--------|
| `%f` | `number` | fixed point — `precision` digits after the point (default 6) |
| `%e` `%E` | `number` | scientific — one digit, `.`, `precision` digits, then `e` or `E` and a signed exponent of at least two digits (default 6) |
| `%g` `%G` | `number` | the shorter of the `%f` and `%e` forms, `%G` spelling the exponent with `E` |

- Rounding is the `std::math::toFixed` rule — `precision` places, halfway
  away from zero — and its overflow note applies to `%f` unchanged
  ([`math.md`](math.md)). An `int` operand renders as its exact `float`
  value.
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
  that reads back as the same `float` — the digits `str` prints — choosing
  the form the same way.
- `%f` and `%e` give six digits after the point when precision is omitted,
  the Go and C default.

```xulo
print(std::fmt::format("%f", [2.5]))           // runs: 2.500000
print(std::fmt::format("%.2f", [3.14159]))     // runs: 3.14
print(std::fmt::format("%e", [1234.5]))        // runs: 1.234500e+03
print(std::fmt::format("%.3g", [0.1]))         // runs: 0.1
print(std::fmt::format("%g", [1e-7]))          // runs: 1e-07
```

## Flags, width, and precision

| Decoration | Meaning | Verbs |
|------------|---------|-------|
| `-` | pad to the right instead of the left | every verb |
| `0` | pad with `0` instead of spaces | `%d` `%x` `%X` `%o` `%b` `%f` `%e` `%E` `%g` `%G` |
| `+` | a positive value carries `+` | the numeric verbs (the same set) |
| ` ` | a positive value's sign position is a space | the numeric verbs (the same set) |
| `#` | the alternate form | `%x` `%X` `%o` `%b` |

- **Width** is a minimum, in Unicode scalar values, for every verb but
  `%%`: a shorter result is padded and a longer one passes through. Padding
  with `0` is sign-aware — sign, then the `#` prefix, then the zeros, then
  the digits — so `%#08x` of `255` is `0x0000ff` and `%08.2f` of `-3.1` is
  `-0003.10`.
- **Precision** is meaningful for `%f`, `%e`, `%E`, `%g`, `%G` (the digits it
  names) and for `%s` and `%q` (the text is cut to that many scalars). On
  any other verb a precision is a runtime failure.
- Two combinations carry the C rules: `0` is ignored when `-` is given, and
  `+` outranks ` `. Every other combination is ordinary.
- The decoration table is closed: a flag outside its verb's column is a
  runtime failure — `%+s` and `%#d` are mistakes, not no-ops.

```xulo
print(std::fmt::format("%-6s|", ["hi"]))       // runs: hi    |
print(std::fmt::format("%+8.2f", [3.1]))       // runs:    +3.10
print(std::fmt::format("%.3s", ["hello"]))     // runs: hel
```

## Failure model

| Kind | Members |
|------|---------|
| Total | none — `format` is defined only for a well-formed template and a matching argument list |
| Runtime failure | `format` — every malformed template (a verb outside the table, a repeated flag, a flag the verb does not accept, a decorated `%%`, a `%` with no verb, a `.` with no precision digits) and every mismatched argument list (a directive count disagreeing with `args`, an operand of the wrong kind for its verb, a precision outside its verb's domain — `0 …= 15` for `%f`/`%e`/`%E`, `1 …= 17` for `%g` — or a `%c` scalar outside `0 …= 0x10FFFF` or in U+D800 … U+DFFF) |
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
std::fmt::format("%d")                          // runs: runtime failure — one directive and no argument
std::fmt::format("%d", ["a"])                   // runs: runtime failure — `%d` takes an int value
std::fmt::format("%z", [1])                     // runs: runtime failure — no verb `z`
std::fmt::format("%d", [1], [2])                // error[E0207]: expected 1 to 2 arguments, found 3
std::fmt::format(42, [1])                       // error[E0202]: expected `string`, found `int`
std::fmt::format<int>("%d", [1])                // error[E0807]: `std::fmt::format` accepts no type arguments
Fmt::format("%d", [1])                          // error[E0607]: `Fmt` lives under `std`: use `std::fmt::format`
```
