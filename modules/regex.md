# `std::regex`

`std::regex` compiles regular-expression patterns and matches them against
strings. Its face is three **types** and no values: the module has no
functions to call and no value namespace to bind
([`README.md`](README.md) § 2.3), so `RegExp::compile` is reached through
the imported type name, and substitution and splitting are methods on
`string` ([`../README.md`](../README.md) § 1.3, item 3). The module is
host-independent: the same pattern, flags, and input give the same matches
on every machine (§ 5.7 of [`README.md`](README.md)). Patterns are ordinary
strings — the language has no `/pattern/` literal.

Bind the types by name; the namespace forms a module with values would have
do not exist:

```xulo
use std::regex::{RegExp, RegexError};  // ok: the canonical form
use std::regex::RegExp;                // ok: one type under its own name
use std::regex::*;                     // ok: RegExp, RegexError, RegexMatch
use std::regex;                        // error[E0607]: no namespace to bind
use std::regex as R;                   // error[E0607]: same
use std::regex::RegExp::compile;       // error[E0607]: members have no children
use std::regex::Pattern;               // error[E0601]: module `std::regex` does not export `Pattern`
```

For the same reason no call is spelled through the module: `RegExp::compile(…)`
is reached through the imported type name, and a value path into the module
resolves no members — `std::regex::find("a", "b")` is `E0601`,
`std::regex::RegExp::compile("a")` is `E0607`, and both carry the guidance
that `std::regex` types are reached through their imported name.

## Types

| Type | Shape | Role |
|------|-------|------|
| `RegExp` | methods `test`, `find`, `find_all` | a compiled pattern |
| `RegexError` | `{ message: string }` | what `RegExp::compile` reports on failure |
| `RegexMatch` | `value: string`, `index: int`, `end: int`, `group(…)` | one match's text, position, and capture groups |

- After the import the names annotate — `fn probe(re: RegExp)`,
  `fn show(m: RegexMatch)`, a `Result<RegExp, RegexError>` signature. All
  three also resolve in an annotation **without** an import — `fn probe(re:
  RegExp)` at file scope needs no `use` — while expressions that name them
  (`RegExp::compile(…)`) need the binding; the reference implementation's
  current gap here is recorded in [`../README.md`](../README.md) § 1.3.
- The names are types, not values. Even after the import, `let f = RegExp`
  is rejected (`E0101`, a type name where a value is required); the only
  expression a type name leads to is the static call `RegExp::compile(…)`.
- **`RegexError` is this specification's one exception to the shared rule
  that a fallible member's `Err` carries the built-in `Error`**
  ([`README.md`](README.md) § 5.1): `compile`'s failure payload is the
  module's own type. It has the same one field, `message: string`, and is
  read the same way — `e.message` in a `Result::Err` arm:

```xulo
match RegExp::compile("[a-z") {
    Result::Ok(_) => print("compiled")
    Result::Err(e) => print(e.message)   // runs: the engine's parse-error text
}
```

## Compiling

| Member | Signature |
|--------|-----------|
| `RegExp::compile` | `fn(pattern: string, flags: string?): Result<RegExp, RegexError>` |

- One or two positional arguments; the omitted `flags` takes the empty
  string — no flags. Zero or three arguments are `E0207`, a non-`string`
  argument is `E0202`, and named arguments are rejected.
- Failure — an unknown flag or a pattern the engine cannot parse — is
  `Result::Err` of `RegexError` (above): data, never a panic and never the
  built-in `Error`.
- Success carries an immutable `RegExp`; every method below reads the same
  compiled pattern. `?` propagates a compile failure like any other
  `Result` error.

### Flags

| Flag | Effect |
|------|--------|
| `i` | case-insensitive matching, by Unicode case folding |
| `m` | multi-line: `^` and `$` also match at line edges |
| `g` | remembered by the pattern: `string.replace` substitutes **every** match instead of the first |
| `u` | accepted and a no-op — Unicode matching is the engine's default and only mode |
| `s` | dotall: `.` also matches a newline (`\n`) |

- The flag string is scanned left to right; the first character outside
  `i g m u s` fails compilation with ``unknown flag `x` `` — a message the
  wrapper owns. An empty flags string (or the omitted argument) compiles
  with no flags.
- `g` changes nothing else: `test`, `find`, and `find_all` behave the same
  with and without it.

### The engine

The engine's observable contract:

- Matching is over Unicode scalar values; a `string` input is always
  well-formed text. `.` matches any scalar except a newline (`\n`) — the
  `s` flag (Flags) lifts that exception.
- At the leftmost position where a match exists, the engine prefers the
  alternative the pattern orders first: `a|ab` finds `a` in `"ab"`.
- **Look-around assertions and back-references are not supported.**
  `(?=…)`, `(?<=…)`, and `\1` are compile-time failures, delivered as
  `Err` of `RegexError` exactly like any malformed pattern.
- Other malformed patterns — an unclosed group or character class, an
  unrecognized escape — fail the same way, carrying the engine's own
  parse-error text (informative, [`../README.md`](../README.md) § 3.5).

```xulo
use std::regex::{RegExp, RegexError};

match RegExp::compile("[a-z]+", "i") {
    Result::Ok(re) => {
        print(re.test("ABC123"))          // runs: true
        print(re.test("123"))             // runs: false
    }
    Result::Err(e) => print(e.message)
}
match RegExp::compile("a", "z") {
    Result::Ok(_) => print("compiled")
    Result::Err(e) => print(e.message)    // runs: unknown flag `z`
}
match RegExp::compile("a.b", "s") {
    Result::Ok(re) => print(re.test("a\nb"))  // runs: true — without `s` it is false
    Result::Err(e) => print(e.message)
}
```

## Matching

| Member | Signature | Meaning |
|--------|-----------|---------|
| `RegExp::test` | `fn(input: string): bool` | whether the pattern matches anywhere in `input` |
| `RegExp::find` | `fn(input: string): RegexMatch?` | the first (leftmost) match, or `null` |
| `RegExp::find_all` | `fn(input: string): List<RegexMatch>` | every non-overlapping match, left to right |

- `find` reports absence as `null` — ordinary data, not a failure
  ([`README.md`](README.md) § 5.2).
- `find_all` returns matches in order, non-overlapping, **regardless of
  `g`**. An empty match is a match: with pattern `a*` the input `"bab"`
  yields three matches — the empty match at the start, `a`, and the empty
  match at the end — and the scan always advances past a match, so the
  list is finite.
- All three are total for every `string` input
  ([`README.md`](README.md) § 5.3).

```xulo
match RegExp::compile("(\\d+)-(\\d+)") {
    Result::Ok(re) => {
        let m = re.find("room 42-7")
        if m != null {
            print(m.value, m.group(1), m.group(2))  // runs: 42-7 42 7
            print(m.index, m.end)                   // runs: 5 9
        } else {
            print("miss")
        }
        for hit in re.find_all("x 42-7 y 8-9") {
            print(hit.value)                        // runs: 42-7, then 8-9
        }
    }
    Result::Err(e) => print(e.message)
}
```

## `RegexMatch`

| Member | Type | Meaning |
|--------|------|---------|
| `RegexMatch.value` | `string` | the matched text — the whole match |
| `RegexMatch.index` | `int` | where the match starts, in scalar indices from the input's start |
| `RegexMatch.end` | `int` | one past the match's last scalar, in the same indices |
| `RegexMatch.group` | `fn(ref: number \| string): string?` | capture group `ref`, 1-based or by name |

- `index` and `end` count **Unicode scalar values** from the start of the
  input — the same index space as `stringSize`, `stringCodeAt`, and
  `stringSlice` — so `stringSlice(input, m.index, m.end)` is exactly
  `m.value`, and `end == index + stringSize(m.value)` always holds. An
  empty match has `index == end`.
- `group` with a **number** is **1-based**: `group(1)` is the first
  parenthesised group. There is no `group(0)` — the whole match is `value`,
  and `group(0)` returns `null`, as do an out-of-range index and a group
  that did not participate in the match (an optional group, e.g. `(a)?b`
  matched against `"b"`).
- `group` with a **string** reads a named group: `(?P<name>…)` captures
  like a numbered one — the name is group *n* in pattern order — and
  `group("name")` returns its text. A string is read as a **name, never as
  an index** — `group("1")` asks for a group *named* `1` — and a name no
  group has returns `null`, the same absence as every other miss. A named
  group's value is also readable through a replacement template (below).
- Read a match behind a null test, exactly as the example above does:
  `m != null` narrows `m` to `RegexMatch`, and `m.value` / `m.group(1)` /
  `m.group("name")` type-check inside the branch
  ([`README.md`](README.md) § 5.2).

## Substitution

Substitution is a method on `string`, not a module function; the language
specification's string section does not carry it yet
([`../README.md`](../README.md) § 1.3, item 3).

| Member | Signature |
|--------|-----------|
| `string.replace` | `fn(pattern: RegExp, replacer: string \| fn(RegexMatch): string): string` |

- The receiver is the text — `text.replace(re, "x")`. There is no
  `std::regex::replace` and no `RegExp.replace`.
- With `g` the substitution covers every match; without it, only the
  first. A pattern that does not match returns the input unchanged.
- A **string** replacer is a template expanded per match:
  - `$1`… and further digits expand to that capture group; `$0` expands
    to the whole match;
  - `$name` and `${name}` expand to the named group `(?P<name>…)`; a name
    with no such group expands to the empty string;
  - `$$` is a literal `$`, and a `$` that begins neither a group reference
    nor `$$` stays literal.
- A **function** replacer receives the `RegexMatch` and returns the
  replacement text.

```xulo
match RegExp::compile("(\\w+)@(\\w+)") {
    Result::Ok(re) => print("ada@example".replace(re, "$2.$1"))  // runs: example.ada
    Result::Err(e) => print(e.message)
}
match RegExp::compile("oo", "g") {
    Result::Ok(re) =>
        print("foo boo".replace(re, fn(m) { "[" + m.value + "]" }))  // runs: f[oo] b[oo]
    Result::Err(e) => print(e.message)
}
```

- **Runtime-failure domains** (`spec/memory-and-runtime.md`): a function
  replacer that returns a non-string stops the program —
  ``replacement function must return a `string`, found number`` — and so
  does a replacer that is neither a string nor a function, and a pattern
  argument that is not a compiled `RegExp`. Within its declared argument
  types, `replace` is total: it returns a `string` for every input.
- The result is a `string`: binding it to another type is `E0201`. A call
  to a name that is not a method of `string` is not a check-time error
  today; it stops the program at runtime (`` `frobnicate` is not a
  function of a string``).

## Splitting

Like substitution, splitting is a method on `string`, not a module function
([`../README.md`](../README.md) § 1.3, item 3).

| Member | Signature |
|--------|-----------|
| `string.split` | `fn(re: RegExp): List<string>` |

- The routing matches `replace`: the receiver is the text, the argument is
  a compiled pattern, and there is neither a `std::regex::split` nor a
  `RegExp.split`.
- The pieces are what the matches cut from the input — the text before the
  first match, between consecutive matches, and after the last — from the
  same left-to-right, non-overlapping scan `find_all` runs, with its empty
  matches and **regardless of `g`**. A match at index 0 gives an empty
  first piece; a match ending at the end gives an empty last piece.
- A pattern that does not match returns a one-element list holding the
  input, the counterpart of `replace` returning the input unchanged.
- Total for its declared argument types ([`README.md`](README.md) § 5.3);
  a pattern argument that is not a compiled `RegExp` is a runtime failure,
  as `replace`'s is.

```xulo
match RegExp::compile("\\s*,\\s*") {
    Result::Ok(re) => print("a , b,c".split(re))   // runs: [a, b, c]
    Result::Err(e) => print(e.message)
}
match RegExp::compile(",") {
    Result::Ok(re) => print("a,b,".split(re))      // runs: [a, b, ] — empty piece after the last `,`
    Result::Err(e) => print(e.message)
}
```

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | `RegExp::compile` — `Result<RegExp, RegexError>`, the module's own error type |
| Total | `RegExp::test`, `RegExp::find`, `RegExp::find_all`; `string.replace` and `string.split` for their declared argument types |
| Absence (`null`) | `find` with no match; `group` at index 0, out of range, not participating, or by a name no group has |
| Runtime failure | `string.replace`'s function replacer returning a non-string; a replacer that is neither string nor function; a `replace` or `split` pattern argument that is not a compiled `RegExp` |

Every match, substitution, and split is host-independent — the same pattern,
flags, and input give the same results everywhere ([`README.md`](README.md)
§ 5.7). Message text is informative
([`../README.md`](../README.md) § 3.5).

## Checked call examples

```xulo
use std::regex::{RegExp, RegexError};

fn probe(re: RegExp) {
    re.sub("x")                                // error[E0102]: no member `sub` on `RegExp`
    let n: number = "a1".replace(re, "#")      // error[E0201]: found `string`, expected `number`
}

RegExp::compile("[a-z]+", "i")                 // ok: Result<RegExp, RegexError>
RegExp::compile("\\d+")                        // ok: flags default to none
RegExp::compile()                              // error[E0207]: expected 1 or 2 arguments, found 0
RegExp::compile("a", "i", "x")                 // error[E0207]: expected 1 or 2 arguments, found 3
RegExp::compile("a", 1)                        // error[E0202]: argument to `RegExp::compile` must be `string`, found `int`
"x".split(3)                                   // error[E0202]: argument to `string.split` must be `RegExp`, found `int`
RegExp::typo("x")                              // error[E0102]: no member `typo` on `RegExp`
let f = RegExp                                 // error[E0101]: a type name where a value is required
```

With no import in scope, the head is rejected with its guidance message,
and no value path through the module exists:

```xulo
RegExp::compile("a+")                          // error[E0101]: `RegExp` belongs to the built-in `std::regex` module
std::regex::RegExp::compile("a")               // error[E0607]: cannot resolve `std::regex::RegExp::compile`
std::regex::find("a", "b")                     // error[E0601]: module `std::regex` does not export `find`
```
