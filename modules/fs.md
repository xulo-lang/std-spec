# `std::fs`

`std::fs` holds the whole-file filesystem operations: read a file as one
string, write a string to a file, probe existence, remove a file, and create
a directory. Every path is a **host path string** — the module follows the
host's path conventions (separator characters, case handling, the characters
a path may contain), which makes it host-dependent (§ 5.5 of
[`README.md`](README.md)); for portable POSIX path *string* semantics that
give the same answer on every machine, see [`path.md`](path.md). Version 0.1
has no directory listing, no open file handles, and no incremental I/O.

The qualified path needs no import:

```xulo
std::fs::write("./note.txt", "hello")   // ok
std::fs::read("./note.txt")             // ok: Result<string, Error>
```

Import forms are the six of [`README.md`](README.md) § 2.2:

```xulo
use std::fs::*;                         // the five functions
use std::fs as fs;
fs::exists("./note.txt")                // ok: an ordinary bool
use std::fs::{read, write};
```

## Reading a file

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::read` | `fn(path: string): Result<string, Error>` | the whole file as one UTF-8 string |

- `read` yields the file's entire contents as a single string, decoded as
  UTF-8. There are no offsets, no partial reads, and no streaming.
- An empty file yields `Ok("")` — the empty string is content, not absence.
  `read` never yields `null`.
- Each of the following is `Err`:
  - the file does not exist,
  - the file cannot be read (permissions, I/O error),
  - the path names a directory, and
  - the file's bytes are not valid UTF-8.

```xulo
match std::fs::read("./empty.txt") {
  Result::Ok(text) => print(text == "")   // runs: true
  Result::Err(e) => print(e.message)
}
```

## Writing a file

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::write` | `fn(path: string, contents: string): Result<unit, Error>` | create or truncate the file, then write `contents` |

- On success the file contains exactly `contents`, encoded as UTF-8 bytes;
  writing the empty string creates (or truncates to) an empty file, which is
  success, not an error.
- If the file already exists it is **truncated to zero length first**: the
  previous content is fully replaced, never merged or appended.
- The file is created when absent, but the parent directory is **not**
  created: a missing parent is `Err` — `mkdir` is the member that makes
  directories.
- Writing to a path that names an existing directory is `Err`.

```xulo
std::fs::write("./log.txt", "old contents that are longer")  // ok
std::fs::write("./log.txt", "new")                           // ok — truncates
// a subsequent read of "./log.txt" yields Ok("new")
```

## Probing existence

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::exists` | `fn(path: string): bool` | whether the path names something |

- `exists` is **total**: it returns a `bool` for every string — never a
  `Result`, never `null`.
- It reports whether *anything* is at the path — a file or a directory —
  following the host's symlink resolution; it does not distinguish the kind
  of thing found.
- A probe that cannot be answered — a permission problem on a parent
  directory, path syntax the host rejects — reads as **`false`**, not as a
  failure: an unanswerable question counts as "not there".
- Absent path → `false`; present file or directory → `true`.

## Removing a file

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::remove` | `fn(path: string): Result<unit, Error>` | delete one file |

- `remove` deletes **one file**. A missing path is `Err`, and a second
  `remove` of the same path is `Err` as well — success means the file is gone.
- A directory path is `Err`: `remove` does not remove directories, empty or
  not; deleting a tree is outside this module.
- Symlink resolution and permission checks follow the host's filesystem;
  this specification pins no cross-host behaviour for them.

## Creating directories

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::mkdir` | `fn(path: string): Result<unit, Error>` | create a directory and its missing parents |

- `mkdir` creates the named directory **and every missing parent** above it
  (recursive creation): one call can bring `a/b/c` into existence.
- Calling it on a path that already exists as a directory succeeds — `Ok`,
  not a failure — so calling it twice is safe.
- When the path exists as a file, or any component of it is a file, the call
  is `Err`.
- The empty path names nothing and is `Err`.
- Existing parents are never an obstacle: only a real one (a file in the
  way, no permission) fails.

Directory enumeration — listing a directory's entries — is not part of
version 0.1.

## A round trip

The pattern a program lives by: write, read back, confirm, remove, confirm
again.

```xulo
let path = "./std_demo.txt"
match std::fs::write(path, "hello std") {
  Result::Ok(_) => match std::fs::read(path) {
    Result::Ok(text) => print(text)            // runs: hello std
    Result::Err(e) => print(e.message)
  }
  Result::Err(e) => print(e.message)
}
print(std::fs::exists(path))                   // runs: true
match std::fs::remove(path) {
  Result::Ok(_) => print("removed")            // runs: removed
  Result::Err(e) => print(e.message)
}
print(std::fs::exists(path))                   // runs: false
```

## Host paths

A path string that names one file on one host may name another — or nothing —
on a different host, and members of this module make no cross-host promises:
the same string can behave differently under Windows and POSIX path rules, or
on a case-insensitive filesystem. [`path.md`](path.md) is the portable
companion: pure string operations with POSIX rules that are identical on
every host, whose results are still strings that `std::fs` then interprets
under the host's conventions.

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | `read` `write` `remove` `mkdir` |
| Total | `exists` |
| Absence (`null`) | none — `read` of an empty file is `Ok("")`, not `null` |
| Runtime failure | none — no `std::fs` member has a runtime-failure argument domain |

Failures are ordinary `Result::Err` values carrying `Error { message }`;
message text is informative ([`README.md`](README.md) § 5.1,
[`../README.md`](../README.md) § 3.5), and the contract is the `Err` shape
plus the failure classes each section above enumerates.

## Checked call examples

```xulo
std::fs::read("notes.txt")                // ok
std::fs::exists("notes.txt")              // ok: an ordinary bool — no Result
std::fs::write("a.txt", "hi")             // ok
std::fs::read(42)                         // error[E0202]: expected `string`, found `int`
std::fs::write("a.txt")                   // error[E0207]: expected 2 arguments, found 1
std::fs::scan(".")                        // error[E0601]: no member `scan`
Fs::read("a.txt")                         // error[E0607]: `Fs` lives under `std`: use `std::fs::read`
```
