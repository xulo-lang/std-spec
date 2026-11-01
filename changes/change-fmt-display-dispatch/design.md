# Design: Text Verbs as a Display/Debug Consumer

## Where the text verbs point

### The Output rendering of `print` — RECOMMENDED

**Decision.** `%v` and `%s` render by exactly the Output rendering the
`print` intrinsic uses: the consult paragraph (a user type asks its
`Display`, then its `Debug`) and the recursive table, cited as
`spec/builtins/intrinsic-functions.md` § Output.

- **It is the chapter's own intent, finally cited correctly.** The chapter
  already claims containers print their elements and `null` prints `null` —
  claims only the Output table supports. The fix is to name the table that
  was always meant.
- **One rendering for the whole language.** `print`, `str`, interpolation,
  and `fmt` would otherwise be three documents' worth of "how a value
  becomes text". The consult and the table are the language's; `fmt` adds
  only the directive.
- **New rows arrive free.** `Array`, tuples, `Result`, `Range`, `Error`,
  `unit`, optionals, `Ptr`, `Format` itself — every type the language gains
  a row for is `%v`-renderable without a chapter edit.

### Continue citing `str`'s table — rejected

`str` accepts base types and `Display` implementors and rejects everything
else at compile time (`E0212`). Citing it while promising container output
is the present inconsistency; narrowing `%v` to `str`'s acceptance would
break `format("%v", [[1, 2]])`, which the chapter has always promised.

### A rendering table owned by `fmt` — rejected

Duplicating the table inside this specification would fork the one thing the
language change just unified, and every new language type would need a
mirror edit here. The document hierarchy (this specification cites the
language's for types) points the same way the `Result` precedent does.

## Precision: handed over, never re-applied

### Hand the precision to the implementation; do not cut after — RECOMMENDED

**Decision.** For a `%v`/`%s` operand whose type is consulted, the
directive's precision (or `null`) and `#` (or `false`) build the `Format`
the implementation receives, and its return is the verb's text. No cut is
applied afterwards.

- The language pins the hand-off: "A consumer hands a directive's precision
  to `display` unchanged" (`spec/types/traits.md`, Built-in formatting
  traits). Cutting after would apply the same request twice, in two
  different units — what the type counted (decimal places, significant
  digits) and what the consumer counts (scalars) — and destroy exactly the
  renderings the contract exists to serve: a `Money` rounding to two decimal
  places returns `"1234.56"`; a post-cut to `.2` would leave `"12"`.
- Layout is the consumer's job and stays there: width, alignment, and
  padding are applied by `format` over the returned text, exactly as over a
  base type's rendering. The split the language change exists for —
  *semantics in the `Format`, layout in the engine* — is this rule.

### Hand it, then cut anyway — rejected

Double-applies the request in incompatible units (above). The contract's
"whatever the type did not lay out, the consumer does" is about layout,
which `Format` does not even carry; precision, once handed over, belongs to
the implementation's reading.

### Cut only; never hand over — rejected

Leaves `Format` unfilled by its own engine: `precision` would exist solely
for direct `v.display(Format(…))` calls, and a type whose whole purpose is
precision-aware text (`Temperature`, `Money`) could never meet a directive.
The contract sentence would have no practicable consumer.

### What a `Debug` return contributes

**Decision.** A `Debug` return is used as returned, in the same way: the
`Format` is handed to it (the signature requires the parameter), it ignores
it in this version (the language says so), and no cut follows.

The alternative — cutting a `Debug` return because it did not read the
precision — would make the rule depend on whether an implementation looked
at its `Format`, which the contract explicitly declines to require. The
stateable rule is: *the cut applies exactly when no implementation was
consulted.*

## The `#` flag on text verbs

### Admit `#` to `%v` and `%s`; pass `alternate` through — RECOMMENDED

**Decision.** `#` joins the `%x` `%X` `%o` `%b` column of the decoration
table for `%v` and `%s`. It sets `Format.alternate: true` for whichever
implementation is consulted; for a value rendered by the table it has no
receiver and no effect.

- `Format.alternate` otherwise has no producer in the language at all: no
  consumer with a directive exists except this engine, and the language's
  own example for the field is "a hex-flavored rendering" of a type that
  means something by it. Opening `#` is how the field earns its place.
- The closed-table discipline survives: `#` is still *outside* the `%d`,
  `%s`-without-`#`… columns it was outside before — `%#d` and `%+s` remain
  runtime failures — and a flag once admitted to a verb's column is a
  property of the template, not of the runtime value, exactly as width is
  today.
- A table-rendered value ignoring `#` is not a no-op flag in the chapter's
  sense: the verb accepts it, the value simply has no implementation to give
  it to. This parallels a width longer than the text passing through.

### Keep `#` failing on `%v` and `%s` — rejected

Would preserve the table untouched but strand `alternate` with no engine
producer: the field the language just added would be reachable only through
hand-written `v.display(Format(alternate: true))` calls, which is a feature
of the contract, not a substitute for the engine the contract was written
for.

### `%#v` selects `Debug` — deferred

A flag whose meaning differs by verb — `%v` asking `Display` first, `%#v`
asking `Debug` on purpose, Rust's `{:#?}` in spirit — is coherent and
richer. It is deferred because it splits `#`'s meaning across the same
column this change is already opening, and because the consult's automatic
fallback to `Debug` already covers the structural case. If an explicit
selector is wanted later, it arrives as its own change with its own
examples.

## Table renderings keep the cut

### One rule: cut iff no implementation was consulted — RECOMMENDED

**Decision.** When the operand renders by the table — base types, `null`,
containers, an `enum` whose type implements neither — a written precision
cuts the rendered text to that many scalars, as the chapter says today.

- It keeps the rule to a sentence, stated once in the dispatch prose.
- It matches Go's `%.3s` of a non-string, and it keeps `format("%.3s",
  ["hello"])` — the chapter's existing example — working unchanged.
- The cut of a container rendering (`%.3s` of `[1, 2, 3]` is `[1,`) is
  strange but defined, and examples will show it once so no reader has to
  guess.

### Fail precision on containers — rejected

Would make the flag's validity depend on the runtime shape of the operand
for no gain: the value is still a valid `%s` operand, the text is still
well-defined, and the chapter's philosophy marks *template* mistakes as
failures, not value-level oddities. Strictness belongs in the decoration
table, which this change does not loosen.

## Fixed-bit operands

### Widen the integer and floating-point verbs — RECOMMENDED

**Decision.** An **integer value** for `%d` `%x` `%X` `%o` `%b` `%c` is an
`int` or a fixed-bit integer (`i8` … `i64`, `u8` … `u64`, `usize`,
`isize`), rendered in the verb's base whatever the operand's type was. The
float verbs take `number`, `f32`, or `f64`; an `f32` is widened to the
`float` it exactly names before the verb applies.

- The Output table already renders every one of these types; refusing them
  here would make `%v` more capable than `%d` on the same value.
- The argument list is `List<any>`: the checker never sees the element
  types, so today's rule only expresses itself as a surprise at run time —
  a `u8` that printed fine under `str`/`print` dying under `%d`.
- Widening `f32` to its exact `float` keeps one *value conversion* and one
  set of formatting rules: the rounding rules, the special values, and the
  precision domains are stated once for `float`, and `f32` inherits them —
  at any written precision the two readings of an `f32` coincide, because
  the widened value *is* the operand's exact value. The one place they can
  disagree is the shortest-roundtrip default of `%g`; see below.

### An integer has no exact `float` value beyond ±2⁵³ — pin the rounding here

**Decision.** An integer operand is converted to the `float` nearest its
value — exact within ±2⁵³, ties to even beyond.

- The chapter's old phrase, "its exact `float` value", is false for most
  `i64` and `u64` values: `9223372036854775807` has no exact `float`
  rendering, and the widening this change introduces makes the case routine
  rather than rare.
- The language does not pin the rounding: `int + float` promotes in
  arithmetic (`spec/type-system/checking-rules.md`), but neither the
  promotion nor any conversion states a tie rule, and coercion refuses the
  jump for values (`spec/type-system/coercion.md` — no coercion rounds).
  A chapter that formats must therefore say what it does; it says ties to
  even (IEEE-754 roundTiesToEven) — the rule the format itself defines for
  rounding a value to a representable one.

### `%g` without precision keeps the operand type's shortest digits

**Decision.** Omitted precision prints the shortest digit string that reads
back as the same value: the `f32` row of the Output table for an `f32`
operand, the `float` row otherwise (integers having been converted as
above). The form rule takes `precision` to be that string's digit count —
the significant digits unique read-back needs — so `0.1` prints `0.1`
(`-1` neither below `−4` nor reaching `1`) and `1e20` prints `1e+20`
(`20` reaching `1`). This is Go's default-precision rule; without it the
form rule's reference to `precision` would dangle whenever the directive
writes none.

- Pure widening for `%g` as well would disagree with `%v` on one and the
  same value: `%v` of an `f32` third prints `0.33333334` (the `f32` row),
  while a widened shortest-roundtrip would print `0.3333333432674408`. The
  two verbs the chapter calls identical would split.
- Only the default diverges. With a written precision `%g` rounds the
  operand's exact value — the same real number either way — so no second
  renderer is introduced: the explicit-precision path is shared with
  `float`, the default path borrows the digits the Output table already
  defines for the type.
- The alternative — a widened default and a note that `%g` may disagree
  with `%v` — was rejected because the chapter's first promise about these
  two verbs is that they are the same thing.

### Keep `int`/`number`; fixed-bit values fail — rejected

Conservative, but it freezes a failure the language's own table has already
answered, and every fixed-bit value in an `any` list would need a conversion
call (`str` does not take them either — another reason the widening belongs
here).

## Failure-model consequences

**Decision.** One new runtime-failure case, in the argument-domain class
the chapter already uses: a `struct` operand of neither `Display` nor
`Debug` under `%v` or `%s`. Everything else in the failure model is
unchanged — malformed templates, argument-count mismatches, wrong-kind
operands, precision domains, `%c`'s scalar domain — with one completion: a
width or precision whose digits do not fit an `int` joins the
malformed-template list, closing a hole the chapter has always had (the
grammar writes `Digit { Digit }` with no bound, and the parser must land
somewhere). The text verbs take every precision an `int` can hold; a cut
longer than the text is a no-op, exactly as a width longer than the text
passes through.

- It is the run-time counterpart of the compile-time `E0212` the same value
  gets from interpolation, `str`, and `print`; the consult table in the
  language says "a compile-time error" where the checker can see the type
  and this chapter says "runtime failure" where it cannot.
- `unknown` has no runtime counterpart: a value in an `any` list always has
  a type, so the language's `unknown` row of the consult never arises here.

## The signature is unchanged

**Decision.** `fn(template: string, args: List<any> = []): string` stands.
This change alters what a directive *does*, not how a call is written:
no variadic parameters, no named arguments into the template, no positional
reuse, no precompiled `Arguments` value. Each of those is either a language
question or a performance question, and none of them is needed for the
dispatch above — the engine fills `Format` from the directive it already
parses.
