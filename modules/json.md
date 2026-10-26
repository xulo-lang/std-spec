# `std::json`

`std::json` holds the three functions that move between JSON text and the
runtime value model: `parse` reads a document into the value it describes,
`stringify` writes a value back as compact or indented JSON text, and
`valid` answers whether a text is a document at all. `parse` and `stringify`
are fallible, `valid` is total, and the module is host-independent with
deterministic output — the same inputs give the same values and the same
text on every machine (§ 5.5 and § 5.7 of [`README.md`](README.md)).

The signatures are written with `any`, the type of a value the checker does
not check: `any` is assignable to and from every type and admits every member
access ([`../README.md`](../README.md) § 1.3, item 2). No `std` member of
version 0.1 takes a type parameter, so `std::json::parse<int>(…)` is rejected
(`E0807`, [`README.md`](README.md) § 4). The qualified path needs no import:

```xulo
std::json::parse("[1, 2, 3]")             // ok: Result<any, Error>
std::json::stringify({ a: 1 })            // ok: Result<string, Error>
std::json::valid("{}")                    // ok: bool
```

Import forms are the six of [`README.md`](README.md) § 2.2; the face is
exactly the three functions ([`README.md`](README.md) § 3):

```xulo
use std::json::*;                         // binds parse stringify valid
use std::json as json;
json::parse("[]")                         // ok: the same member
use std::json::{parse, stringify, valid};
```

## The value model

`parse` maps JSON onto the runtime values, and `stringify` maps them back:

| JSON | Runtime value |
|------|---------------|
| `null` | `null` |
| `true` / `false` | a `bool` |
| `string` | a `string` |
| `number` | a `number` — binary64 |
| `array` | a `list`, elements in order |
| `object` | an `object` with the same members |

- **Numbers are binary64.** A value within ±2⁵³ round-trips exactly; beyond
  that a parse-then-stringify round trip MAY change the decimal
  representation — `9007199254740993` can come back as `9007199254740992`.
  JSON has no spelling for `NaN` or the infinities: a document containing
  those tokens is a syntax error, and such values only arise from
  computation ([`math.md`](math.md)).
- **Object members are ordered by UTF-8 byte order in both directions.**
  `parse` builds the object's members in that order, so the input
  document's key order is not preserved; duplicate keys in the input — the
  last occurrence wins. `stringify` emits members in that same sorted order
  (see below).
- **Nesting is bounded.** Both directions share one ceiling — 128 levels of
  arrays and objects. A document or a value nested deeper than the ceiling
  is `Err`, never a partial result and never a trap.
- **Lists and tuples are arrays.** A `list` serializes with its order
  intact; a positional tuple serializes as the array of its elements —
  `(1, "x")` becomes `[1,"x"]`. `parse` reads an array as a `list`: the
  tuple's type is not carried by the text, and the consumer rebuilds it —
  the plain shape a struct comes back in too ([`Structs`](#structs)).
- **Maps are objects.** A `Map<string, V>` and an object literal emit
  their entries as members under the sorted-members rule above. Keys of
  other types are **textified** on the way out: an `int` by its decimal
  form, a finite `float` by the number form `stringify` prints (so `1.0`
  keys as `1`), a `bool` as `true` or `false`. Two keys that textify to
  the same string are `Err`; so are a non-finite `float` key and an enum
  key — an enum has no JSON spelling here any more than it does as a
  value. Textified keys come back as `string` keys from `parse`.
- **Sets are arrays, in a defined order.** A `Set<T>` has no order of its
  own — iteration over one is unspecified (the language specification) —
  so `stringify` supplies one: the elements are emitted **sorted by their
  compact JSON text in UTF-8 byte order**, one rule across every
  hashable element type and the same kind of choice that sorts object
  members (§ 5.7). The order is on the text, not the value: a set of
  numbers orders by their digits — `10` before `2`. `parse` reads a
  `list`; `setFromList` restores the set — its elements are distinct, so
  the trip keeps every one.

```xulo
std::json::stringify(setFromList(["b", "a"]))   // runs: ["a","b"] — ordered by element text
std::json::stringify(Map<int, string>{1: "x"})  // runs: {"1":"x"} — the key textified
std::json::stringify((1, "x"))                  // runs: [1,"x"] — the tuple's elements
match std::json::parse("{\"b\": 2, \"a\": 1}") {
  Result::Ok(v) => match std::json::stringify(v) {
    Result::Ok(text) => print(text)          // runs: {"a":1,"b":2}
    Result::Err(e) => print(e.message)
  }
  Result::Err(e) => print(e.message)
}
```

## Parsing

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::json::parse` | `fn(text: string): Result<any, Error>` | the single JSON value `text` describes |

- One value per document: leading whitespace and trailing whitespace around
  it are fine, but any other trailing content is `Err` — and so are the
  empty string and whitespace-only text.
- Each of the following is `Err` (message text is informative
  ([`../README.md`](../README.md) § 3.5); the classes are the contract):
  - a **syntax error** — an invalid token or a malformed structure, with a
    position in the message,
  - **unexpected end of input** — an empty or truncated document,
  - **trailing content** after the top-level value,
  - a **number outside binary64 range** — `1e400` is `Err`, never an
    infinity,
  - **nesting past the depth ceiling** (above).
- There is no size limit beyond the depth ceiling: a large but shallow
  document parses.
- Success yields `Ok(value)`, typed `any`. The checker does not look inside:
  what the program does with the result — member access, indexing,
  arithmetic — is its own business, and `any` admits it all.
- `parse` never uses `null` as a stand-in for failure: the `null` it can
  yield is JSON's `null`, inside `Ok`.

```xulo
match std::json::parse("{") {
  Result::Ok(v) => print("parsed")           // not reached
  Result::Err(e) => print(e.message)         // runs: invalid JSON: …
}
```

## Stringifying

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::json::stringify` | `fn(value: any, indent: number? = null): Result<string, Error>` | the JSON text of `value` — compact, or indented when `indent` is given |

- The default — `indent` omitted, `null`, or `0` — is a single compact line
  with no insignificant whitespace: the only separators are `,` between
  elements and members and `:` after a key — `{"a":[1,"two"],"b":2.5}`.
- **`indent` requests pretty output.** A value in **`1 …= 8`** replaces the
  compact line with one where every array and object opens on its own line,
  each nesting level indented `indent` spaces relative to its parent, `: `
  (colon and one space) after a key, and items separated by `\n` — and an
  empty object or array stays on one line as `{}` or `[]`. The text ends
  with the value: **no trailing newline**.
- The `indent` domain is **`0 …= 8`**, in spaces per level: a fractional
  value counts as its truncation toward zero, and `0` is the compact line
  above. A value outside the domain — negative, above 8, non-finite — is
  `Err`, checked **before** the value is walked, so a bad `indent` is
  reported whatever the value holds. The domain is an `Err`, never a
  runtime failure.
- **Indentation never disturbs determinism.** Sorted members, escaping, and
  number forms are shared with the compact output, so the same value gives
  the same text at every indent.
- **Members are emitted sorted by UTF-8 byte order** — deterministic, and
  never the order the value was built in (the determinism claim of § 5.7 of
  [`README.md`](README.md)).
- **Numbers** print with no needless fractional part: an integral value
  prints as an integer — `2.0` prints `2`, so an integer-valued `number`
  round-trips without a `.0`, and a negative zero prints as `0`. A
  non-integral value prints in the shortest decimal form that restores the
  same binary64 (`0.1` stays `0.1`; a third prints
  `0.3333333333333333`), and a magnitude too large for plain digits may
  use scientific notation (`1e+21`).
- **Text** is escaped only where JSON requires it: `\"`, `\\`, `\n`, `\r`,
  `\t`, `\b`, `\f`, and `\u00XX` for the other control characters. Every
  other character is emitted raw UTF-8 — non-ASCII text is not `\uXXXX`
  escaped.
- `stringify(null)` yields the four characters `null`.
- Each of the following is `Err`:
  - **an `indent` outside `0 …= 8`** — checked before the walk (above).
  - **`NaN` and the infinities** — they have no JSON spelling (the message
    names the value: ``cannot serialize NaN as JSON``). This includes a
    `NaN` or infinity nested at any depth inside lists and objects
    (`std::math::NAN` is [`math.md`](math.md)'s).
  - **Values JSON cannot represent**: closures and other functions, enum
    values, foreign pointers (`Ptr`), signals, and promises. Ordinary
    objects — object literals, struct-shaped values — and lists serialize
    as JSON objects and arrays (what a struct's members are is
    [`Structs`](#structs)'s; the collection rules — lists, tuples, maps,
    sets — are [`The value model`](#the-value-model)'s).
  - **Map keys that do not textify distinctly**: two keys naming the
    same string, a non-finite `float` key, an enum key — the key rules of
    [`The value model`](#the-value-model).
  - **Cycles** — a list or object that contains itself. A cycle is caught
    by the depth ceiling and arrives as `Err` (a depth message naming the
    ceiling), never as a trap or a hang.
  - **Nesting past the depth ceiling**, cycle or not.

```xulo
match std::json::stringify({ name: "x", n: 3 }) {
  Result::Ok(text) => print(text)            // runs: {"n":3,"name":"x"}
  Result::Err(e) => print(e.message)
}
std::json::stringify({ name: "x" }, 2)
// runs:
// {
//   "name": "x"
// }
match std::json::stringify(std::math::NAN) {
  Result::Ok(text) => print(text)
  Result::Err(e) => print(e.message)         // runs: cannot serialize NaN as JSON
}
```

## Structs

A struct value is an ordinary member set for `stringify`: **every field the
value carries becomes a member of the JSON object — private fields
included.** `pub` is a compile-time name rule (the language specification,
`struct` chapter), not part of a value: the object that reaches
`stringify` carries its fields and nothing that could mark one private:

```xulo
struct User { name: string, age: int, passwordHash: string }

let alice = User(name: "Alice", age: 30, passwordHash: "5e88…")
std::json::stringify(alice)
// runs: {"age":30,"name":"Alice","passwordHash":"5e88…"} — every field, private included
```

Two explicit roads lead out of that default when a program wants a
different set of members:

- **Hand over the members.** Construct the object to serialize, field by
  field:

```xulo
std::json::stringify({ name: alice.name, age: alice.age })
// runs: {"age":30,"name":"Alice"}
```

- **Or implement `toJson`.** A struct MAY declare an inherent method named
  `toJson` that yields the value to serialize — an ordinary method, `pub`
  when callers live in another module — and the program passes its result
  on:

```xulo
impl User {
  pub fn toJson(self): any { ({ name: self.name, age: self.age }) }
}

std::json::stringify(alice.toJson())        // the text of that object
std::json::stringify(alice.toJson(), 2)     // the same members, indented
```

Neither road is wired into `stringify`: **the program calls `toJson`,
never `stringify`** — method resolution is module-scoped (the language
specification, `traits` chapter), so a member inside `std` cannot see a
program's `impl`, and nothing in `std::json` looks for the name. What
`toJson` yields obeys `stringify`'s existing rules: a `NaN`, a closure, or
a cycle inside it arrives as the same `Err` as anywhere else.

The way back is manual by the same design: `parse` yields the plain object
the document describes — never a struct — and the constructor restores the
type, the `any` value admitting every field access (§ 1.3, item 2 of
[`../README.md`](../README.md)):

```xulo
match std::json::parse(text) {
  Result::Ok(v) => User(name: v.name, age: v.age)    // the type comes from the constructor
  Result::Err(e) => print(e.message)
}
```

A struct therefore does not round-trip into itself: what
`parse(stringify(u))` carries is the members, typed `any`.

## Validating

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::json::valid` | `fn(text: string): bool` | whether `text` is a well-formed JSON document |

- **Total**: every text answers — `true` or `false`, never a `Result`,
  never `null`, never a runtime failure.
- `valid(text)` is `true` **exactly when** `parse(text)` yields `Ok`. The
  two share one grammar, one depth ceiling, and one binary64 range: trailing
  content, an empty document, a number outside binary64, or nesting past
  128 levels answers `false` here and `Err` there — the answers never
  disagree.
- What the text describes is not part of the question: `valid` settles
  whether it parses; the value is `parse`'s business.

```xulo
std::json::valid("{\"a\": 1}")     // runs: true
std::json::valid("{\"a\": }")      // runs: false
std::json::valid("")               // runs: false — an empty document
```

## Round trips

- For every value JSON can represent whose numbers fit within ±2⁵³,
  `parse(stringify(v))` reproduces `v`, and the text is stable after one
  trip: `stringify(parse(stringify(v)))` equals `stringify(v)`. Nominal
  types are the one exception to the first half: a struct, a tuple, or a
  set comes back as its plain shape — object, list, list — and a map with
  textified keys comes back with `string` keys; the constructor or
  `setFromList` restores the type ([`Structs`](#structs)). The **text**
  still stabilizes in every case.
- **Indentation is not part of the round trip.** `parse` does not see it,
  and `stringify` with `indent` omitted always returns the compact text —
  `parse(stringify(v, 2))` reproduces `v` exactly as `parse(stringify(v))`
  does.
- **Key order is not part of the round trip.** Input order is discarded at
  `parse` (members are built in UTF-8 byte order), and `stringify` always
  emits that order — so the reordering is observable: the round-tripped
  value's `stringify` shows the sorted order, and its text is identical to
  the first `stringify`'s.

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | `parse`, `stringify` |
| Total | `valid` |
| Runtime failure | none — no `std::json` member stops the program on the program's own inputs (`stringify`'s `indent` domain is an `Err`, not a runtime failure) |

Both failures are ordinary `Result::Err` values carrying `Error { message }`;
the module introduces no error type of its own ([`README.md`](README.md)
§ 5.1). Message text is informative ([`../README.md`](../README.md) § 3.5).
A `parse` failure is about the **text**; a `stringify` failure is about the
**value**.

## Checked call examples

```xulo
std::json::parse("[1, 2]")                 // ok
std::json::stringify("hi")                 // ok — the JSON text of a string includes its quotes
std::json::stringify({ a: 1 }, 2)          // ok: indented text
std::json::valid("{}")                     // ok: an ordinary bool — no Result
std::json::parse("{")                      // runs: Result::Err — a syntax error
std::json::stringify({ a: 1 }, 9)          // runs: Result::Err — indent outside 0 …= 8
std::json::parse(42)                       // error[E0202]: argument to `std::json::parse` must be `string`, found `int`
std::json::valid(42)                       // error[E0202]: argument to `std::json::valid` must be `string`, found `int`
std::json::stringify()                     // error[E0207]: expected 1 argument, found 0
std::json::loads("{}")                     // error[E0601]: `std::json` has no member `loads`
Json::parse("{}")                          // error[E0607]: `Json` lives under `std`
std::json::parse<int>("[]")                // error[E0807]: `std::json::parse` accepts no type arguments
```
