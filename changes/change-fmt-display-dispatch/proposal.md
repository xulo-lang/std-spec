# Change fmt's Text Verbs to a Display/Debug Consumer

**Status:** review · **Date:** 2026-11-01 · **Affects:** `modules/fmt.md`

## Summary

The text verbs of `std::fmt` — `%v` and `%s` — become a consumer of the
language's formatting contracts. Where the chapter today says they render by
"exactly the table the `str` intrinsic uses", they render by exactly the
Output rendering the `print` intrinsic uses: a user-declared type consults
its `Display` first and its `Debug` when no `Display` exists, under a
`Format` built from the directive, and every other value is rendered by the
Output table. A written precision becomes that `Format`'s `precision`; a
written `#` becomes its `alternate`. The `str` table is no longer cited,
because `str` rejects containers at compile time while `%v` has always meant
to print them.

Alongside the dispatch, the integer and floating-point verbs widen their
operands to the fixed-bit numeric types the language now has — `i8` … `i64`,
`u8` … `u64`, `usize`, `isize`, `f32`, `f64` — which the Output table
already renders.

```xulo
#Derive(Debug)
struct Point { x: int, y: int }

struct Money { cents: int }

impl Display for Money {
  fn display(self, format: Format): string {
    std::math::toFixed(self.cents / 100.0, format.precision ?? 2)
  }
}

std::fmt::format("%v", [Point(x: 1, y: 2)])     // ok: "Point(x: 1, y: 2)"
std::fmt::format("%.1s", [Money(cents: 12345)]) // ok: "123.5"
std::fmt::format("%v", [[1, [2, 3]]])           // ok: "[1, [2, 3]]"
```

Until this change is accepted, `modules/fmt.md` is normative and the deltas
in [`specs/`](specs/) are the text the chapter gains once the change is
accepted.

## Motivation

### The chapter cites a table that rejects its own examples

`fmt.md` says `%v` and `%s` render by "exactly the table the `str`
intrinsic uses", and in the same breath says "a container prints its
elements". The language disagrees on both counts: `str` accepts only base
types and `Display` implementors, and rejects a `List`, a `Map`, a `null`,
or a `struct` with no implementation at compile time (`E0212`). The table
that renders containers — the one with the `List`, `Map`, `Set`, `null`, and
`<function>` rows — is the Output table under `print`
(`spec/builtins/intrinsic-functions.md`), and the chapter's own examples
stand on that side: `format("%t / %v", [true, null])` is `"true / null"`,
which `str(null)` could never produce. The citation names the wrong
intrinsic, and what it should name has grown a consult paragraph and a
dozen new rows since the chapter was written.

### `Format` exists to be filled by this engine

The language change that replaced `ToString` with `Display`, `Debug`, and
`Format` (accepted in the language specification on 2026-11-01) was designed
with this module as its engine: "`Format` carries semantics only … layout is
applied by the consumer"; "a consumer hands a directive's precision to
`display` unchanged". A directive that asks for `.2` and meets a type that
means something by precision is precisely the case the contract was written
for — and today's chapter, which cuts the rendered text to two scalars and
never calls anyone, cannot meet it. `std::fmt` is the consumer the contract
names; the chapter should say so.

### Fixed-bit values already have a rendering

The Output table renders `i8` … `u64`, `usize`, `isize` as decimal numerals
and `f32` as its shortest reading. The integer verbs take an `int` and the
float verbs take `number`, so a `u8` fed to `%d` is a runtime failure today
— a value the language can hold, the table can render, and the program
naturally reaches through an `any` argument list. The operands should follow
the table.

The widening corrects two claims the chapter already gets wrong. "An `int`
operand renders as its exact `float` value" is false beyond ±2⁵³ —
`9223372036854775807` has no exact `float` rendering — and the language
pins no rounding for the jump, so the chapter must state one (nearest,
ties to even). And `%g` without precision must print the operand type's
shortest digits — the Output table's `f32` row for an `f32` — or it would
disagree with `%v`, the verb it is defined to be identical to, on one and
the same value.

## Scope

**In:**

- the dispatch of `%v` and `%s` — the Output consult (Display, then Debug)
  and the Output table, cited as such;
- the `Format` built from a directive — `precision` from the precision,
  `alternate` from `#` — handed to whichever implementation is asked, with
  the returned text used as returned;
- `#` admitted to the `%v` and `%s` columns of the decoration table;
- integer and floating-point verbs over fixed-bit values, with the
  integer→`float` conversion pinned (nearest, ties to even) and `%g`'s
  default digits taken from the operand's own type;
- the failure-model row for a `struct` with neither implementation under
  `%v` or `%s`, and the completion that a width or precision whose digits
  do not fit an `int` is a malformed template;
- examples.

**Out:**

- new members — `format` remains the module's whole face;
- variadic or named arguments, positional reuse (`%[2]d`), width from the
  argument list (`*`);
- a verb that selects `Debug` on purpose (a `%?`-shaped idea) — the consult
  already falls back to it, and an explicit selector can come later;
- `Format` growth — the language's optional-field policy owns that;
- `std::io` integration (`write`-shaped members) — downstream of the
  Read/Write traits of the roadmap;
- a precompiled-template type — performance, not capability.

## Language-specification dependency

The adoption depends on a change that has **already landed** in the language
specification: `change-tostring-to-display` (accepted 2026-11-01,
xulo-lang/spec @ `f885d74`) — `Display`, `Debug`, and `Format` in the
prelude, the Output consult paragraph, and the completed rendering table.
No further language change is required: this proposal adds no syntax and no
diagnostic code.
