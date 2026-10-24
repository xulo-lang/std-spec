# `std::json`

`std::json` holds the two functions that move between JSON text and the
runtime value model: `parse` reads a document into the value it describes,
`stringify` writes a value back as compact JSON text. Both members are
fallible, and the module is host-independent with deterministic output — the
same inputs give the same values and the same text on every machine (§ 5.5
and § 5.7 of [`README.md`](README.md)).

The signatures are written with `any`, the type of a value the checker does
not check: `any` is assignable to and from every type and admits every member
access ([`../README.md`](../README.md) § 1.3, item 2). No `std` member of
version 0.1 takes a type parameter, so `std::json::parse<int>(…)` is rejected
(`E0807`, [`README.md`](README.md) § 4). The qualified path needs no import:

```xulo
std::json::parse("[1, 2, 3]")             // ok: Result<any, Error>
std::json::stringify({ a: 1 })            // ok: Result<string, Error>
```

Import forms are the six of [`README.md`](README.md) § 2.2; the face is
exactly the two functions ([`README.md`](README.md) § 3):

```xulo
use std::json::*;                         // binds parse and stringify
use std::json as json;
json::parse("[]")                         // ok: the same member
use std::json::{parse, stringify};
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

```xulo
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
| `std::json::stringify` | `fn(value: any): Result<string, Error>` | the compact JSON text of `value` |

- The output is a single compact line with no insignificant whitespace: the
  only separators are `,` between elements and members and `:` after a key —
  `{"a":[1,"two"],"b":2.5}`.
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
  - **`NaN` and the infinities** — they have no JSON spelling (the message
    names the value: ``cannot serialize NaN as JSON``). This includes a
    `NaN` or infinity nested at any depth inside lists and objects
    (`std::math::NAN` is [`math.md`](math.md)'s).
  - **Values JSON cannot represent**: closures and other functions, enum
    values, foreign pointers (`Ptr`), signals, and promises. Ordinary
    objects — object literals, struct-shaped values — and lists serialize
    as JSON objects and arrays.
  - **Cycles** — a list or object that contains itself. A cycle is caught
    by the depth ceiling and arrives as `Err` (a depth message naming the
    ceiling), never as a trap or a hang.
  - **Nesting past the depth ceiling**, cycle or not.

```xulo
match std::json::stringify({ name: "x", n: 3 }) {
  Result::Ok(text) => print(text)            // runs: {"n":3,"name":"x"}
  Result::Err(e) => print(e.message)
}
match std::json::stringify(std::math::NAN) {
  Result::Ok(text) => print(text)
  Result::Err(e) => print(e.message)         // runs: cannot serialize NaN as JSON
}
```

## Round trips

- For every value JSON can represent whose numbers fit within ±2⁵³,
  `parse(stringify(v))` reproduces `v`, and the text is stable after one
  trip: `stringify(parse(stringify(v)))` equals `stringify(v)`.
- **Key order is not part of the round trip.** Input order is discarded at
  `parse` (members are built in UTF-8 byte order), and `stringify` always
  emits that order — so the reordering is observable: the round-tripped
  value's `stringify` shows the sorted order, and its text is identical to
  the first `stringify`'s.

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | `parse`, `stringify` |
| Total | none |
| Runtime failure | none — no `std::json` member stops the program on the program's own inputs |

Both failures are ordinary `Result::Err` values carrying `Error { message }`;
the module introduces no error type of its own ([`README.md`](README.md)
§ 5.1). Message text is informative ([`../README.md`](../README.md) § 3.5).
A `parse` failure is about the **text**; a `stringify` failure is about the
**value**.

## Checked call examples

```xulo
std::json::parse("[1, 2]")                 // ok
std::json::stringify("hi")                 // ok — the JSON text of a string includes its quotes
std::json::parse("{")                      // runs: Result::Err — a syntax error
std::json::parse(42)                       // error[E0202]: argument to `std::json::parse` must be `string`, found `int`
std::json::stringify()                     // error[E0207]: expected 1 argument, found 0
std::json::loads("{}")                     // error[E0601]: `std::json` has no member `loads`
Json::parse("{}")                          // error[E0607]: `Json` lives under `std`
std::json::parse<int>("[]")                // error[E0807]: `std::json::parse` accepts no type arguments
```
