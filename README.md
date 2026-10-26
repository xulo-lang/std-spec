# Xulo Standard Library Specification

**Status:** Draft
**Version:** 0.1.0
**Date:** 2026-10-24
**Language:** English

This repository contains the normative specification of the Xulo standard
library: the built-in `std` root, its ten modules, every member's signature,
and the semantics a conforming implementation must provide.

## 1. Purpose and Scope

This specification defines the *design* of the standard library:

- how `std` is reached — the namespace root, qualified paths, and the `use`
  forms a module has,
- the ten modules of version 0.1 and the complete face of each,
- the signature and the meaning of every member: total or fallible, which
  failures are values and which are runtime failures,
- the shared conventions the modules hold in common: the `Result<T, Error>`
  contract, absence as `null`, units, portability, and blocking.

The design rests on one claim: *a library failure is data, never control
flow.* A missing file, a malformed document, a refused connection, an unset
variable — each arrives as `Result::Err` or as `null`, in a shape the caller
can `match` or propagate with `?`. The standard library never throws, never
catches, and (except where a chapter states a runtime failure for an argument
domain) never stops the program for the program's own inputs.

It is written for language and library implementers, tool authors, and
advanced users who need a precise account of what each member accepts, what
it yields, and what it does when the world does not cooperate.

### 1.1 Out of scope

The following are deliberately **not** specified here:

| Excluded | Where it is documented |
|----------|------------------------|
| The language itself: the prelude types and operations, `Result`/`Error`/`Task`, the intrinsics `print`, `println`, `str`, `assert`, the `Task` namespace, name resolution, typing, diagnostics | [Xulo language specification](https://github.com/xulo-lang/spec) |
| The prelude namespace spellings `Math::` and `Time::` — being folded into `std::math`/`std::time` by a language change (see § 1.3) | language specification, `spec/builtins/intrinsic-functions.md` (pending change) |
| Library modules not among the ten — the blueprint inventory of the website (collections, async, sync, thread, HTTP, `Random`, `Instant`, `DateTime`, …) and the `std::io` generic direction | § 1.2 Roadmap |
| Tutorials, guides, and narrative reference prose | [xulo-website](https://xulo.org) |
| The implementation: the `xulo-std` crate, the checker's registered tables, the CLI, the compiled back ends | implementation repository (`xulo`) |
| Package registry protocol, manifest format, tooling | xulo-website ecosystem documentation |

**Self-hosting note (informative).** The standard library supplies the
services a self-hosted tool needs — reading files, writing to standard error,
reading arguments, choosing an exit status
(`modules/fs.md`, `modules/io.md`, `modules/process.md`) — so a self-hosted
implementation in Xulo reaches them here rather than through the foreign
boundary of `ffi.md` in the language specification.

### 1.2 Roadmap

The modules below are **candidates for future changes, not part of version
0.1** — most named by the xulo-website blueprint, one recording a design
direction. Each enters this specification through the workflow in
[`changes/`](changes/README.md) when — and only when — it is being
implemented:

| Candidate | Source | Status |
|-----------|--------|--------|
| `std::collections` — `HashMap`, `BTreeMap`, `HashSet`, `Deque` | website `standard-library-net-fs` | not specified |
| `std::async` / `std::sync` / `std::thread` — intervals, channels, atomics, barriers | website `standard-library-net-fs` | not specified |
| `std::net` HTTP surface — `fetch`, `HttpListener`, `UdpSocket` | website `standard-library-net-fs` | not specified |
| `std::fs` object surface — `File`, `Path`, `FileMode` | website `standard-library-net-fs` | not specified |
| `std::io` generic surface — `Read`/`Write` traits over streams, files, sockets | design direction (Rust `std::io`, Go `io.Reader`) | not specified |
| `std::math` extras — `Random`, `Matrix`, `Vector`, `Complex` | website `standard-library-net-fs` | not specified |
| `std::time` extras — `Instant`, `Duration`, `DateTime` | website `standard-library-net-fs` | not specified |
| `std::process` extras — `Command` builder | website `standard-library-net-fs` | not specified |

### 1.3 Language-specification dependencies

This specification is written for the world the implementation already
implements (xulo-lang/xulo @ `08f6556`, including commit `238ff42`, *Math/Time
moved into the std tree*). Three language changes must land in
[xulo-lang/spec](https://github.com/xulo-lang/spec) to make the two documents
consistent; until they do, this specification governs the ten modules and the
language specification governs everything in § 1.1's first row.

1. **`Math`/`Time` move out of the prelude.** The language specification still
   lists `Math` and `Time` as prelude namespaces with their own signature
   tables. The accepted direction is that the standard library owns them:
   the qualified forms `std::math::…` and `std::time::…` are the only
   spellings this specification defines, and the language change shrinks
   `spec/builtins/` to the `Task` namespace plus `print`, `println`, `str`,
   and `assert`, drops `Math`/`Time` from the prelude tables and the
   reserved-name list, and updates the prose that names them elsewhere (the
   narrowing sentence in `spec/type-system/coercion.md`, for example).
2. **The type `any`.** `std::json` signatures are written with `any`, the type
   of a value the checker does not check: `any` is assignable to and from
   every type and admits every member access — distinct from the language's
   top type `unknown`, which admits nothing without narrowing. The language
   specification does not yet name `any`; adopting it is a language change.
3. **`string.replace`.** The substitution method on `string` that takes a
   `RegExp` is specified in [`modules/regex.md`](modules/regex.md); the
   language specification's string section does not yet mention it.
4. **A never-returning call in statement position.** `std::process::exit(1)`
   is an ordinary expression statement; the language's `E0218` row admits
   only excepted control-flow constructs and discarded `send` tasks, so that
   row must admit a never-returning call when `std::process` lands.

**Convergence items** (contract here vs. the reference implementation today —
work for the compiler, not for this document):

| Item | This specification | Implementation today |
|------|--------------------|----------------------|
| Numeric signatures — `float` results (the constants, `sqrt`, `sin`, …), `int` results (`floor`, `ceil`, `round`, `trunc`), `u64` (`sleep`, the clocks, the `rotate*` amount) | as the chapters write them | the checker reports `number` for every numeric result and checks numeric parameters against `number` (the loose numeric layer); the compiled path puts the four rounding results in an F64 slot rather than an integer one |
| Head and member diagnostics — capital head (`Json::parse(…)`), unknown member (`std::json::dumps(…)`), called constant (`std::math::PI(…)`) | `E0607` / `E0601` / `E0204` | reported as the coarse fallback `E0003`, without the fine-grained code |
| `std::regex` reachability — unimported types in annotation position, aliased types (`use std::regex::RegExp as R;` then `R::compile(…)`) | all three types resolve in annotations without an import; an alias behaves like any other binding; the diagnostics the tables state | only `RegexMatch` resolves unimported; an aliased type does not answer `::compile`; regex path rejections report the coarse `E0003` |
| `std::net` member spelling | `tcp_listen` etc. as registered | matches — but is snake_case against the lowerCamelCase naming convention; settle before 1.0 |

## 2. Status and Versioning

This document is a **draft**. A chapter that exists is intended to be
complete and internally consistent, but the specification as a whole has not
been frozen.

- **Minor version** increments add members or clarify meaning without
  removing or reworking them.
- **Major version** increments may remove members or change signatures and
  semantics.
- Proposed changes are tracked under [`changes/`](changes/README.md) in the
  same OpenSpec-style workflow the language specification uses; accepted
  changes are folded into the module chapters and archived.

**Compatibility.** The two specifications version independently; each version
of this document is written against a stated language specification version:

| std spec | language spec | reference implementation |
|----------|---------------|--------------------------|
| 0.1.x | 0.1.x (draft, [xulo-lang/spec](https://github.com/xulo-lang/spec)) | xulo-lang/xulo @ `08f6556` |

## 3. Normative Conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this
document are to be interpreted as described in RFC 2119 and RFC 8174 when, and
only when, they appear in all capitals.

Additional conventions:

1. **Code examples are non-normative.** They illustrate rules; where an
   example conflicts with the prose, the prose wins.
2. **Xulo source** is shown in fenced blocks tagged `xulo`.
3. **Examples state their expected outcome.** A block that illustrates a rule
   whose result is not obvious from the surrounding prose SHOULD mark it in a
   trailing comment: `// ok` for a program the checker accepts,
   `// error[E0202]` for one it rejects with that code, `// error` where the
   failure has no code, and `// runs: …` where the runtime result is what the
   example is about. Diagnostic codes belong to the language specification
   (`spec/type-system/errors.md`); this document uses them, never invents
   them.
4. **A rule that admits one form and forbids another shows both**, marking
   the accepted form `// ok` and the rejected one with its error code, so the
   implementation repository can lift a block into its test set unchanged.
5. **Error message text is informative.** A message may be reworded at any
   time; the contract of a fallible member is its `Result` shape, the
   distinction between `Err` and an absence, and the *classes* of failure a
   chapter enumerates. Programs and tests MUST NOT match on message strings
   unless a chapter explicitly pins one.
6. **Type notation.** `T?` denotes the optional type, `T | U` a union,
   `fn(A): R` a function type, `Task<T>` an asynchronous computation,
   `Result<T, E>` the built-in result, and `{ f: T, … }` a structural object
   type (the notation `spec/concurrency.md` uses for a service handle).
   `never` marks a call that returns no value at all — `std::process::exit`
   is the only such member — which the checker types `null` so the call
   composes in statement position.
7. **Citations.** `spec/<path>` names a file of the
   [Xulo language specification](https://github.com/xulo-lang/spec); links
   without that prefix are files of this repository.
8. **This specification adds no syntax.** Grammar belongs to the language
   specification; a change here that needs new syntax must land there first.

## 4. Document Map

| Document | Contents |
|----------|----------|
| [`modules/README.md`](modules/README.md) | The `std` root: inventory, reachability, module faces, shared conventions, well-formedness table |
| [`modules/math.md`](modules/math.md) | `std::math` — constants, float and integer functions, `random` |
| [`modules/time.md`](modules/time.md) | `std::time` — wall and monotonic clocks, sleep |
| [`modules/io.md`](modules/io.md) | `std::io` — the process streams: writes, flush, reads, terminal probes |
| [`modules/fs.md`](modules/fs.md) | `std::fs` — whole-file read/write/append, timestamps, exclusive creation, inspection, copy/rename, existence, removal, directories, tree walk, temporaries |
| [`modules/path.md`](modules/path.md) | `std::path` — portable POSIX path string operations |
| [`modules/os.md`](modules/os.md) | `std::os` — environment, working directory, process identity |
| [`modules/net.md`](modules/net.md) | `std::net` — TCP sockets over opaque numeric handles |
| [`modules/process.md`](modules/process.md) | `std::process` — exit status, argv, child capture |
| [`modules/json.md`](modules/json.md) | `std::json` — parse, stringify, and validate the JSON value model |
| [`modules/regex.md`](modules/regex.md) | `std::regex` — `RegExp`, `RegexMatch`, and substitution |
| [`signature-index.md`](signature-index.md) | Every member in one alphabetical index |
| [`changes/`](changes/README.md) | Incremental change proposals (OpenSpec style) |

## 5. Terminology

Terms are used consistently across chapters; the definitions below are
normative for the whole specification.

| Term | Definition |
|------|------------|
| **absence** | The state a fallible-or-optional member reports instead of a failure: the `null` inside `T?`. End of stream, unset variable, no match — absence is ordinary data, never an `Err`. See [`modules/README.md`](modules/README.md). |
| **blocking member** | A member whose call does not return until the host operation completes — in version 0.1 the `std::io` readers, the `std::net` openers and reads/writes, and `std::process::run`. See [`modules/README.md`](modules/README.md). |
| **face** | The set of names a module's glob binds — `use std::math::*` binds the face. A module may export a name that its face withholds (see `std::io`). |
| **fallible** | Of a member that returns `Result<T, Error>`: its failure is a value in the `Err` slot. |
| **handle** | An opaque numeric name for a host resource — a socket — issued only by a `std::net` opener, accepted only by `std::net` members, and invalid after `sock_close`. |
| **member** | One constant or function of a module, selected with `::` (`std::math::PI`, `std::fs::read(…)`), or bound by a `use`. |
| **std root** | The built-in namespace head `std`: provided by the compiler, backed by no file, no manifest, and no package initialization. |
| **total** | Of a member that never fails: it returns its declared type directly, with no `Result` and no absence, for every admissible argument. |
| **runtime failure** | A condition that stops the program — a panic, as `spec/memory-and-runtime.md` defines it. This specification uses it only for argument domains a chapter states explicitly (for example `std::math::toFixed`'s `digits`). |

## 6. Relationship to Other Documents

```
this specification   normative: what std is and each member means
        ▲
        │ explains / teaches
xulo-website docs    narrative reference and tutorials (partly stale, see below)
        ▲
        │ demonstrates
implementation       checker tables, `xulo-std` crate, back ends
        ▲
        │ implements
language spec        normative for the language around std
```

- **The language specification** ([xulo-lang/spec](https://github.com/xulo-lang/spec))
  is normative for everything in § 1.1's first row — types, diagnostics,
  `use` grammar, `Result`/`Error`. Where this document cites a diagnostic
  code, that document assigns it.
- **The implementation repository** (`xulo`) is *non-normative*: it may lag
  or lead this specification. § 1.3 lists the known gaps.
- **xulo-website content** is narrative. Its EN stdlib pages predate this
  specification in the following points and are superseded by it; its ZH pages
  describe an earlier exception-based design and are superseded entirely.

| Website says | This specification says |
|--------------|-------------------------|
| `use json as JSON;`, `use regex::{…}` (bare heads) | `use std::json as JSON;`, `use std::regex::{…}` — bare module heads are rejected |
| `Math::PI`, `Time::now()` in scope without `use` | `std::math::PI`, `std::time::now()` — qualified through `std`, or imported |
| `JSONError` enum, `JSON::parse<T>`, `parse_async` | failures are `Result<_, Error>`; there is no `JSONError` and no `parse_async` |
| clocks return `u64`, `Time::now_nanos` casing | `u64`, lowerCamelCase `nowNanos` (agrees on type, differs on namespace) |
| `File`, `fetch`, `HashMap`, `Random`, … | roadmap candidates only (§ 1.2) |
| `string.replace` availability unstated | specified in [`modules/regex.md`](modules/regex.md) |

## 7. Contributing

Chapters are edited as plain Markdown. Proposed changes must land under
[`changes/`](changes/README.md) before they are folded into the module
chapters; the workflow, artifacts, and index are described there. When a
change needs language syntax or a new diagnostic code, the language
specification is changed first, and the change here cites it.

## 8. License

Everything in this repository — the specification prose and the Xulo source
examples — is covered by a single license, the MIT License, whose full text
is in [`LICENSE`](LICENSE).
