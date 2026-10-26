# The `std` Root

`std` is a built-in namespace root: the compiler provides it, it is backed by
no file, no manifest entry, and no package initialization, and every program
reaches it without declaring anything. The head `std` has exactly one level
of children — the **modules** — and each module owns a set of members
(constants, functions, or types) selected with `::`.

Version 0.1 defines ten modules. The set is closed: a path whose module is
not one of the ten does not resolve (`E0607`), and a member a module has not
listed below does not exist (`E0601`). New modules and new members enter
through the workflow in [`../changes/`](../changes/README.md).

This chapter owns how the library is *reached* and what its modules hold in
common. Each member's meaning is specified in its own module chapter.

## 1. Module inventory

| Module | What it covers | Chapter |
|--------|----------------|---------|
| `std::math` | mathematical constants; float and integer functions; random draws | [`math.md`](math.md) |
| `std::time` | wall and monotonic clocks; sleep; ISO 8601 text and calendar fields | [`time.md`](time.md) |
| `std::io` | the process streams: writes, flush, reads, terminal probes | [`io.md`](io.md) |
| `std::fs` | whole-file read/write/append; timestamps; exclusive creation; inspection; copy and rename; existence; removal; directories and tree walk; temporaries | [`fs.md`](fs.md) |
| `std::path` | portable POSIX path string operations | [`path.md`](path.md) |
| `std::os` | environment; working directory; host directories; process identity | [`os.md`](os.md) |
| `std::net` | TCP sockets over opaque numeric handles | [`net.md`](net.md) |
| `std::process` | exit status; argv; child capture | [`process.md`](process.md) |
| `std::json` | parse, stringify, and validate the JSON value model | [`json.md`](json.md) |
| `std::regex` | `RegExp`, `RegexMatch`, substitution, and splitting | [`regex.md`](regex.md) |

## 2. Reaching the standard library

### 2.1 Qualified paths

The qualified form needs no `use`:

```xulo
std::io::println("hello")
std::math::floor(3.7)                 // ok
let z = std::math::sqrt(2.25)         // ok: 1.5
```

Three rules bind the head:

1. **`std` is a namespace, not a value.** Using `std`, or a module path with
   no member — `std::math` — as a value or a callee is rejected (`E0607`).
   Only `std::<module>::<member>` denotes something.
2. **A namespace is not a value either.** `let m = std::math;` is rejected
   (`E0607`). Reach members through a path or a `use` binding.
3. **A module member may be constant or function.** `std::math::PI` reads as
   an ordinary `float` value; `std::math::floor(3.7)` as an ordinary call.

### 2.2 `use` forms

Six forms bind `std` names. Each binds under a **local name** — by default
the last segment, or the name after `as`:

| Form | Binds | Example |
|------|-------|---------|
| `use std::<module>;` | the module's namespace under `<module>` | `use std::math;` — then `math::floor(3.7)` |
| `use std::<module> as <alias>;` | the module's namespace under `<alias>` | `use std::io as io;` — then `io::print("x")` |
| `use std::<module>::<name>;` | one member | `use std::math::floor;` — then `floor(3.7)` |
| `use std::<module>::{a, b};` | a group of members | `use std::io::{flush, eprint};` |
| `use std::<module>::*;` | the module's **face** (§ 3) | `use std::math::*;` — the constants and 37 functions |
| `use type std::<module>::<Type>;` | a name for types only | `use type std::regex::RegExp;` |

Notes:

- A namespace binding reached through its alias accepts the same member
  selection as the qualified path — `io::print(…)` and `std::io::print(…)`
  denote the same member — including `std::io`'s withheld `print`/`println`
  (§ 3) and the glob form through the alias: `use m::*` on `m = std::math`
  binds the same face as `use std::math::*`.
- Binding a name the program already declared, or a second binding of one
  name, is a collision (`E0104`).
- An unused binding warns (`W0102`).
- A type-only binding used as a value or callee is rejected (`E0606`).

### 2.3 `use` forms this specification rejects

| Rejected | Code | Because |
|----------|------|---------|
| `use std;` | `E0607` | the root has no members of its own; bind a module, a member, or the face |
| `use std::regex;` · `use std::regex as R;` | `E0607` | `std::regex` has no value namespace; import its names — `use std::regex::{RegExp, RegexError}` |
| `use std::foo;` (unknown module) | `E0607` | the ten modules are the whole tree |
| `use std::math::floor::x;` (too deep) | `E0607` | members have no children |
| `use std::math::nope;` | `E0601` | `module `std::math` does not export `nope`` |
| `use std::io::print;` · `use std::io::{print};` | `E0601` | `print` is a prelude intrinsic name and cannot be rebound by a `use` |
| `use std::io as print;` (any local name that is `print`, `println`, `str`, or `assert`) | `E0601` | same refusal; alias under another name |

`std::io`'s `print`/`println` are the case behind the last two rows: the
bare prelude names are intrinsics of the language, and a `use` binding under
those names would type-check as the member while the runtime resolves the
intrinsic. The qualified path and a namespace alias reach the same member
without the ambiguity.

### 2.4 Shadowing: which `math` wins

For a bare head the resolver tries, in order: **bindings in scope** (namespaces
and aliases from `use`), **manifest packages**, the **root modules of the
program**, and only then the **built-in `std` tree**. Two consequences:

- A program that declares a package or root module named `math`, `json`,
  `net`, … shadows the bare spelling, and `math::floor(…)` is legal *for that
  program* — it names their module, not `std::math`. The shadow does not
  change `std::math::…`.
- A program that declares a package named `std` shadows the built-in root
  entirely; such a program does not reach the standard library through
  `std::`.

### 2.5 Spellings that are rejected

`std` members are named only through `std`. Against an unshadowed program,
these are errors (`E0607`), each hinting at the qualified spelling:

```xulo
math::floor(3.7)         // error[E0607]: `math` lives under `std`: use `std::math::floor`
json::parse("[]")        // error[E0607]: `json` lives under `std`: use `std::json::parse`
Math::PI                 // error[E0607]: `Math` lives under `std`: use `std::math::PI`
Time::now()              // error[E0607]: `Time` lives under `std`: use `std::time::now`
Json::parse("[]")        // error[E0607]: `Json` lives under `std`: use `std::json::parse`
Fs::read("a.txt")        // error[E0607]: `Fs` lives under `std`: use `std::fs::read`
```

Both the bare module heads and the capital namespace spellings are pre-`std`
spellings this specification does not define; the migration from the prelude
`Math`/`Time` namespaces is tracked in
[`../README.md`](../README.md) § 1.3. The `Task` namespace and the intrinsics
`print`, `println`, `str`, `assert` are **not** std members — they stay with
the prelude, and `Task::…` remains valid spelling.

A user type, enum, or trait with one of these names takes the spelling for
itself first, exactly as in § 2.4: the rejection is for heads that resolve to
nothing else.

## 3. Module faces

The table lists each module's **exported** members; the **face** — the set
of names its glob binds — coincides with the exports for nine modules, and
`std::io` withholds two of its ten (see the note below).

| Module | Constants | Functions | Types |
|--------|-----------|-----------|-------|
| `std::math` | `PI` `E` `TAU` `INFINITY` `NAN` | `abs` `min` `max` `sign` `sqrt` `cbrt` `pow` `hypot` `clamp` `floor` `ceil` `round` `trunc` `toFixed` `sin` `cos` `tan` `asin` `acos` `atan` `atan2` `log` `log2` `log10` `log1p` `exp` `expm1` `random` `randomInt` `wrappingAdd` `wrappingSub` `wrappingMul` `popCount` `countLeadingZeros` `countTrailingZeros` `rotateLeft` `rotateRight` | — |
| `std::time` | — | `now` `nowNanos` `monotonic` `monotonicNanos` `sleep` `format` `parse` `parts` `fromParts` `weekday` | — |
| `std::io` | — | `print` `println` `eprint` `eprintln` `flush` `readLine` `read` `readN` `stdinIsTTY` `stdoutIsTTY` | — |
| `std::fs` | — | `read` `write` `append` `createNew` `touch` `setTimes` `exists` `stat` `copy` `rename` `remove` `mkdir` `readDir` `walk` `removeDir` `tempFile` `tempDir` | — |
| `std::path` | — | `isAbsolute` `join` `dirname` `basename` `split` `ext` `stem` `withExt` `normalize` `relative` | — |
| `std::os` | — | `getenv` `setenv` `cwd` `chdir` `tmpDir` `homeDir` `pid` `platform` `arch` `hostName` `userName` `cpuCount` | — |
| `std::net` | — | `tcp_listen` `tcp_accept` `tcp_connect` `sock_read` `sock_write` `sock_shutdown` `sock_close` `sock_local_addr` `sock_peer_addr` | — |
| `std::process` | — | `exit` `args` `run` | — |
| `std::json` | — | `parse` `stringify` `valid` | — |
| `std::regex` | — | — | `RegExp` `RegexError` `RegexMatch` |

**`std::io` exports ten functions and withholds two from its face.**
`print` and `println` are members — `std::io::print(…)` is a checked, typed
call to the flushing stream write ([`io.md`](io.md)) — but they are withheld
from the glob face and from single/group `use`, because their local names are
the prelude intrinsics (`E0601`, § 2.3). Reach them through the qualified
path or a namespace alias:

```xulo
std::io::print("io: ")           // ok: the flushing write
use std::io as io;
io::println("hello stdout")       // ok: same member, aliased
use std::io::*;                   // ok: eprint eprintln flush readLine read readN stdinIsTTY stdoutIsTTY
// print("hi") after the glob is the prelude intrinsic — different buffering
// (see io.md), and `use std::io::print;` is error[E0601]
```

**`std::regex` holds types and no values.** Its three types are visible to
the checker without an import — annotation positions resolve them — while
*using* them as expressions (`RegExp::compile(…)`) requires a binding:
`use std::regex::{RegExp}` or `use std::regex::*`. That is why its namespace
form is refused (§ 2.3): there is no value namespace to bind.

## 4. Checking a member call

A call to a `std` member is checked exactly like a call to any other
function, against the signature this specification gives it:

- **Arity** is fixed by the signature; too few or too many arguments are
  `E0207`. Exactly two members accept an optional trailing argument —
  `std::process::run` and `RegExp::compile` (1 or 2 arguments; the omitted
  argument takes the default their chapters state).
- **Argument types** are the signature's parameter types; a non-assignable
  argument is `E0202`. Literals coerce as the language's coercion table
  allows (`spec/expressions/literals.md`).
- **Explicit type arguments** are never accepted: `std::json::parse<int>(…)`
  is `E0807`. No `std` member has a type parameter in version 0.1.
- **Constants are values, not callables**: `std::math::PI(…)` is rejected
  (`E0204`, *not callable*).
- **Unknown members** are `E0601`: ``module `std::json` has no member
  `dumps` `` — the same code a `use` of a non-member gets.
- **A capital or bare head** that reaches a `std` member is `E0607`
  (§ 2.5), even when the member exists.

### Well-formedness summary

| Rule | Code | Example |
|------|------|---------|
| `std` or a module path used as a value or callee | `E0607` | `let s = std;` · `std::math(1)` |
| bare module head; capital namespace head (unshadowed) | `E0607` | `fs::read("a")` · `Math::PI` |
| unknown module under `std` | `E0607` | `use std::foo;` · `std::foo::x()` |
| unknown member under `std` | `E0601` | `std::json::dumps(…)` · `use std::math::nope;` |
| a `std::regex` type in expression position with no binding | `E0101` | `RegExp::compile("a")` with no `use std::regex::…` (see [`regex.md`](regex.md)) |
| `use` binding under a prelude intrinsic name | `E0601` | `use std::io as print;` |
| `use std;` · `use std::regex;` | `E0607` | see § 2.3 |
| type-only binding used as a value | `E0606` | `use type std::math::floor;` then `floor(3.7)` |
| two bindings of one name; glob overlapping a declaration | `E0104` | `use std::math::floor;` in a module with `fn floor` |
| wrong argument count | `E0207` | `std::math::min(1, 2, 3)` |
| non-assignable argument | `E0202` | `std::fs::read(42)` |
| explicit type arguments on a `std` call | `E0807` | `std::json::parse<int>("[]")` |
| constant called | `E0204` | `std::math::PI(2.0)` |
| unused `use` | `W0102` | `use std::path::join;` with no use of `join` |
| module-scope declaration bearing a prelude reserved name (language rule; applies to any file) | `E0106` | `fn print(…)` |

Member names are **not** reserved: a module MAY declare `fn read`, `fn
normalize`, or `struct RegExp`; such a declaration stands for its own module
and does not conflict with `std` — only the prelude's four intrinsic names
are reserved for declarations.

## 5. Shared conventions

These hold in every module chapter; a chapter that deviates says so
explicitly.

### 5.1 Failures are values

Every fallible `std` member returns the built-in `Result<T, Error>` — with
one exception: `std::regex::RegExp::compile` returns that module's own
`RegexError` ([`regex.md`](regex.md)). No member throws, catches, or panics
on the program's inputs (chapters state the argument domains that are
runtime failures). Failures arrive in the `Err` slot and are handled by
`match`, by `?`, or by ignoring the value:

```xulo
fn readConfig(path: string): Result<string, Error> {
  match std::fs::read(path) {
    Result::Ok(text) => Result::Ok(text)
    Result::Err(e) => Result::Err(e)      // or: std::fs::read(path)?
  }
}

match std::fs::write("./a.txt", "hello std") {
  Result::Ok(_) => null
  Result::Err(e) => std::io::eprintln(e.message)
}
```

`Error` carries one field, `message: string`. Message text is informative
([`../README.md`](../README.md) § 3.5): the contract is the `Err` shape and
the failure *classes* a chapter enumerates.

### 5.2 Absence is `null`

A member whose result may simply *not exist* — end of stream, an unset
variable, no match, a relation no string can name — returns `T?` and yields
`null` for that case. Absence is data, not failure, and a member never
reports it as `Err`:

| Absence | Member |
|---------|--------|
| end of input | `std::io::readLine()` → `null`; `std::io::readN(max)` → `null` |
| no match | `std::regex::RegExp.find` → `null`; `RegexMatch.group` → `null` for group 0, out of range, a non-participating group, or an unknown name |
| no relative path | `std::path::relative` → `null` when one side is absolute and the other relative, or when `from` would climb above the shared anchor |
| variable not set | `std::os::getenv` → `Ok(null)` (set but the operation failed → `Err`) |

### 5.3 Total, fallible, and runtime failure

- **Total** members return their type directly, for every admissible
  argument — `std::path::join`, `std::math::sqrt`, `std::os::pid`.
- **Fallible** members return `Result<T, Error>` — `std::fs::read`,
  `std::net::tcp_connect`, `std::json::parse`.
- **Optional** members return `T?`, or `Result<T?, Error>` when they are both.
- A **runtime failure** — a condition that stops the program, as
  `spec/memory-and-runtime.md` defines it — occurs only where a chapter
  states the argument domain (for example `std::math::toFixed`'s `digits`
  range). Chapters mark those domains explicitly.

### 5.4 Units

Clocks and delays count in whole units named by the member, never by its
type: `now`, `monotonic`, and `sleep`'s argument are milliseconds; members
whose name ends in `Nanos` are nanoseconds; `std::time::now` counts
milliseconds since the Unix epoch. A unit is part of a member's documented
meaning, not part of its type.

### 5.5 Portability

| Portable (semantics fixed across hosts) | Host-dependent |
|-------------------------------------------|----------------|
| `std::math`, `std::time`, `std::json`, `std::regex`, `std::path` | `std::fs`, `std::os`, `std::net`, `std::process`, `std::io` |

`std::path` is the rule's illustration: it is portable **string** semantics
with POSIX rules regardless of the host, while `std::fs` follows the host's
path conventions. A path built portably with `std::path` and interpreted by
`std::fs` therefore agrees on POSIX hosts and may diverge elsewhere — a
chapter says so where it matters.

### 5.6 Synchronous execution

Every member of version 0.1 is synchronous: the call returns when the host
operation completes. The members that can wait on the outside world for an
unbounded time — the `std::io` readers (`readLine`, `read`, `readN`), the `std::net`
openers and reads/writes, and `std::process::run` — are the **blocking
members** named in the terminology table; while one waits, the caller does
not make progress. Asynchronous
variants are a roadmap candidate (`std::async`, [`../README.md`](../README.md)
§ 1.2), not part of this version. The single member returning a task is
`std::time::sleep`, whose `Task<unit>` must be awaited in an `async` body;
discarding it is `E0218`.

### 5.7 Determinism

Where a chapter claims it, a conforming implementation produces the same
result for the same inputs on every host: `std::json`'s output order, every
`std::path` member, every `std::regex` match, and `std::time`'s calendar
members (`format`, `parse`, `parts`, `fromParts`, `weekday`) are
deterministic.
Transcendental `std::math` functions (`sin`, `log`, `exp`, …) delegate to the
host's math library and MAY differ from another implementation in the last
bits; the classification rules (NaN, infinity), rounding, and wrapping
members are fixed by their chapters.
