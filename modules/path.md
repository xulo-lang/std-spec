# `std::path`

`std::path` holds pure string operations on `/`-separated paths: joining,
testing for an absolute path, taking the parent, the final component, its
stem and extension, splitting those apart, replacing an extension, and
normalizing away redundancy. Every member is total — `""`, all slashes, and
a lone `..` each have a defined answer — so the module has no `Result`, no
absence, and no runtime failure. It is the portable companion to
[`std::fs`](fs.md): it applies POSIX rules on every host, while `fs` follows
the host's conventions ([`README.md`](README.md) § 5.5).

The qualified path needs no import:

```xulo
std::path::join("var/log", "app.log")    // ok: a single string
print(std::path::basename("/etc/hosts"))  // runs: hosts
```

Import forms are the six of [`README.md`](README.md) § 2.2:

```xulo
use std::path::*;               // every function of the face
use std::path as p;
p::normalize("a//b/./c/")       // ok: "a/b/c"
use std::path::{join, ext};
```

## Domain

Each member is a pure string operation over POSIX path syntax. Four rules
bind the whole module:

- **The separator is `/` alone.** A backslash is an ordinary character,
  never a separator: `join("a", "b\\c")` yields `"a/b\\c"`, and no member
  treats `\` specially.
- **Trailing slashes are ignored** before a component is taken (the
  implementation's module doc likens this to Go's `path/filepath`), so
  `"a/b/"` has final component `"b"` and `"dir.d/"` has extension `".d"`.
- **Nothing is resolved.** No member reads the host filesystem, follows a
  symlink, or maps `.` and `..` onto host directories — they stay ordinary
  string components until `normalize` collapses them.
- **Results are identical on every host**, exactly as
  [`README.md`](README.md) § 5.7 requires. A program that must be
  host-independent builds its paths here and hands them to `fs` (see
  *Relationship to `std::fs`* below).

## Absolute

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::path::isAbsolute` | `fn(path: string): bool` | whether `path` names an absolute location |

- The test is the first character alone: a leading `/` — the one separator
  this module recognizes — makes the path absolute, and nothing else can:
  `"a/b"`, `""`, `".."`, and a host-style `"C:\\x"` are all relative.
- It never consults the host: absolute here means "written with a root" in
  POSIX syntax, not "exists" or "resolves" (Domain's third rule).

```xulo
print(std::path::isAbsolute("/etc/hosts"))   // runs: true
print(std::path::isAbsolute("etc/hosts"))    // runs: false
let bare = std::path::isAbsolute("")         // ok: false
```

## Joining

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::path::join` | `fn(a: string, b: string): string` | `b` appended to `a` |

- An absolute `b` — one starting with `/` — **replaces `a` entirely**; the
  result is `b` alone, mirroring Go's `Path::join` and `os.path.join`.
- An empty `a` contributes no separator: the result is `b` itself.
- An `a` ending in `/` is concatenated with `b` with no extra separator;
  otherwise the result is `a + "/" + b`.
- `join` never collapses `.` or `..` — that is `normalize`'s job — and a
  missing `b` would be a checker error, not an empty join: `join("a", "")`
  is `"a/"`, because the empty `b` still follows the separator rule.

```xulo
print(std::path::join("var/log", "app.log"))  // runs: var/log/app.log
print(std::path::join("a/", "b"))             // runs: a/b
print(std::path::join("", "b"))               // runs: b
print(std::path::join("a", "/b"))             // runs: /b
print(std::path::join("a", ""))               // runs: a/
let empty = std::path::join("", "")           // ok: ""
```

## Parent and name

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::path::dirname` | `fn(path: string): string` | everything before the last `/`, trailing slashes ignored |
| `std::path::basename` | `fn(path: string): string` | the final component, trailing slashes ignored |
| `std::path::split` | `fn(path: string): { dir: string, base: string, ext: string }` | `dirname`, `basename`, and `ext` of `path` in one record |

- All members here trim trailing slashes first and then look for the
  last `/`.
- `dirname` yields `""` when the path names no parent — `"a"` and `""` —
  and `"/"` both for a name directly under the root (`"/a"`) and for the
  root itself (`"/"`, `"///"`).
- `basename` yields `""` when the path names no component: `""`, `"/"`,
  and `"///"`.
- The three fields of `split` are exactly the three members' answers for
  the same input — `dir` is `dirname`'s, `base` is `basename`'s, `ext` is
  the `ext` member's below — one call and one binding instead of three.
  Edge inputs follow the three: `split("/")` is `{ dir: "/", base: "",
  ext: "" }`, and `split("a")` is `{ dir: "", base: "a", ext: "" }`.

```xulo
print(std::path::dirname("/a/b/c.txt"))   // runs: /a/b
print(std::path::dirname("a/b/"))         // runs: a
print(std::path::dirname("/a"))           // runs: /
let none = std::path::dirname("a")        // ok: "" — there is no parent
let root = std::path::dirname("/")        // ok: "/" — the root is its own parent
print(std::path::basename("/a/b/"))       // runs: b
print(std::path::basename("file"))        // runs: file
let bare = std::path::basename("/")       // ok: "" — no component is named
let s = std::path::split("/var/log/app.log")
print(`${s.dir} ${s.base} ${s.ext}`)      // runs: /var/log app.log .log
```

## Extension

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::path::ext` | `fn(path: string): string` | the extension of the final component, including the dot |
| `std::path::stem` | `fn(path: string): string` | the final component without its extension |
| `std::path::withExt` | `fn(path: string, ext: string): string` | `path` with the final component's extension replaced |

- The result **includes the dot**: `"a.txt"` yields `".txt"`, and only the
  last dot counts, so `"a.tar.gz"` yields `".gz"`.
- `ext` yields `""` when there is no extension. A dot at the start of the
  name is a hidden-file marker, not an extension (`".bashrc"` → `""`), and
  a trailing dot carries no extension text (`"file."` → `""`).
- The extension is taken from the basename, so a dot in a directory
  component never counts (`"dir.d/f"` → `""`); trailing slashes are already
  gone by then, so `"dir.d/"` yields `".d"`.
- `stem` is `basename` minus exactly what `ext` reports:
  `"archive.tar.gz"` → `"archive.tar"`, `"file"` → `"file"`, and — a
  leading dot being a marker, not an extension — `".bashrc"` →
  `".bashrc"`. When `ext` is `""` nothing is removed, so `"file."` →
  `"file."`.
- `withExt` replaces that extension: the argument takes **one optional
  leading dot** — `".md"` and `"md"` name the same extension — and the new
  final component is the `stem` plus the extension (the `stem` alone when
  the extension argument is empty, which is how an extension is removed:
  `withExt("report.txt", "")` → `"report"`). A name with no extension
  gains one (`"notes"` → `"notes.md"`), hidden files included
  (`".bashrc"` → `".bashrc.md"`). Only the final component changes: the
  parent directories are untouched, no other normalization happens, and
  trailing slashes follow the module rule (`withExt("dir.d/", ".txt")` →
  `"dir.txt"`).

```xulo
print(std::path::ext("report.pdf"))        // runs: .pdf
print(std::path::ext("archive.tar.gz"))    // runs: .gz
let hidden = std::path::ext(".bashrc")     // ok: ""
let trailing = std::path::ext("file.")     // ok: ""
let dirDot = std::path::ext("dir.d/f")     // ok: ""
print(std::path::ext("dir.d/"))            // runs: .d
print(std::path::stem("archive.tar.gz"))   // runs: archive.tar
let hiddenStem = std::path::stem(".bashrc")   // ok: .bashrc
print(std::path::withExt("report.txt", ".md"))   // runs: report.md
print(std::path::withExt("report.txt", "md"))    // runs: report.md — the dot is optional
print(std::path::withExt("report.txt", ""))      // runs: report — the extension is removed
let added = std::path::withExt("notes", ".md")   // ok: notes.md
```

## Normalization

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::path::normalize` | `fn(path: string): string` | `path` with duplicate slashes, `.` components, and resolvable `..` components collapsed |

- Duplicate slashes collapse to one, `.` components disappear, and `..`
  pops the component before it. Trailing slashes are removed.
- A `..` with nothing left to pop is **dropped at the root** of an absolute
  path (`"/../a"` → `"/a"`) but **kept in a relative path** — it may exist
  once directories appear (`"../a/../b"` → `"../b"`, and a `..` that pops
  past the start of a relative path is kept the same way).
- A result with nothing left is `"."` for a relative input and `"/"` for an
  absolute one. Leading double slashes collapse to a single `/`, and the
  only result that ends in `/` is the root `"/"` itself.

```xulo
print(std::path::normalize("a//b/./c/"))     // runs: a/b/c
print(std::path::normalize("/a/b/../../c"))  // runs: /c
print(std::path::normalize("/../a"))         // runs: /a
print(std::path::normalize("../a/../b"))     // runs: ../b
print(std::path::normalize("a/.."))          // runs: .
print(std::path::normalize(""))              // runs: .
print(std::path::normalize("///"))           // runs: /
```

### Relationship to `std::fs`

`std::fs` takes raw path strings and follows the host's conventions;
`std::path` is portable string semantics over `/`. A program that must be
host-independent builds and splits its paths here, then passes the resulting
strings to [`std::fs`](fs.md) — and the host then interprets them by its own
rules. The two modules agree on one host, but `std::path` alone does not
predict what `fs` will do on another
([`README.md`](README.md) § 5.5).

### Totality (whole module)

No `std::path` member returns `Result` or `null`, touches the filesystem, or
stops the program on the program's own inputs: every degenerate argument —
`""`, an all-slash path, a lone `..`, an absolute right operand to `join` —
has a defined string answer. The nine results are fixed for every input and
identical on every host
([`README.md`](README.md) § 5.7).

## Failure model

| Kind | Members |
|------|---------|
| Total | every member — `isAbsolute`, `join`, `dirname`, `basename`, `split`, `ext`, `stem`, `withExt`, `normalize` |
| Runtime failure | none |
| Fallible (`Result`) | none |
| Absence (`null`) | none |

## Checked call examples

```xulo
std::path::join("a", "b")                 // ok
std::path::join("a")                      // error[E0207]: expected 2 arguments, found 1
std::path::join("a", 1)                   // error[E0202]: expected `string`, found `int`
std::path::withExt("a")                   // error[E0207]: expected 2 arguments, found 1
std::path::stem(1)                        // error[E0202]: expected `string`, found `int`
Path::join("a", "b")                      // error[E0607]: `Path` lives under `std`: use `std::path::join`
```
