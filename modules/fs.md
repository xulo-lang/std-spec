# `std::fs`

`std::fs` holds the whole-file filesystem operations: read a file as one
string or one list of bytes, write, append, touch, set its time or
permission bits, create a file exclusively, probe existence, resolve,
inspect, copy, and remove a file, create, list, walk, and remove a
directory, rename a file or a directory, and make a temporary file or
directory. Every path is a **host path string** — the module follows the
host's path conventions (separator characters, case handling, the
characters a path may contain), which makes it host-dependent (§ 5.5 of
[`README.md`](README.md)); for portable POSIX path *string* semantics that
give the same answer on every machine, see [`path.md`](path.md). Version
0.1 has no open file handles and no incremental I/O.

Text is the default: every string here is UTF-8 and `read` refuses bytes
that are not valid UTF-8. Content that is not text has its own members —
`readBytes`, `writeBytes`, and `appendBytes` carry a `List<u8>` — so an
object file or an image crosses this module as bytes, never as a decoded
string (the shared convention is § 5.8 of [`README.md`](README.md)).

The qualified path needs no import:

```xulo
std::fs::write("./note.txt", "hello")   // ok
std::fs::read("./note.txt")             // ok: Result<string, Error>
```

Import forms are the six of [`README.md`](README.md) § 2.2:

```xulo
use std::fs::*;                         // the twenty-four functions
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
- The byte twin is [`readBytes`](#byte-content): the same file and the same
  path failures, with no decoding and so no encoding class.

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

## Appending to a file

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::append` | `fn(path: string, contents: string): Result<unit, Error>` | add `contents` to the end of the file, creating it when absent |

- `append` never truncates: what the file held is kept, and `contents` is
  added after it. `write` is the truncating member; `append` is the growing
  one — a log line wants the second.
- The file is created when absent, with `contents` as its whole content; a
  missing parent directory is `Err` — `mkdir` is the member that makes
  directories.
- Appending the empty string succeeds and changes nothing: the file ends up
  exactly as it was.
- Appending to a path that names an existing directory is `Err`.

```xulo
std::fs::write("./log.txt", "first\n")     // ok — the file holds "first\n"
std::fs::append("./log.txt", "second\n")   // ok — it holds "first\nsecond\n"
```

## Byte content

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::readBytes` | `fn(path: string): Result<List<u8>, Error>` | the whole file as raw bytes |
| `std::fs::writeBytes` | `fn(path: string, bytes: List<u8>): Result<unit, Error>` | create or truncate the file, then write `bytes` |
| `std::fs::appendBytes` | `fn(path: string, bytes: List<u8>): Result<unit, Error>` | add `bytes` to the end of the file, creating it when absent |

- The three are the byte twins of `read`, `write`, and `append`: the same
  path rules and path failures, the same truncate-versus-grow behaviour.
  The difference is the unit, and that nothing is decoded in either
  direction, so every byte sequence survives the round trip.
- `readBytes` yields one element per byte, each in `0 …= 255`; an empty
  file is `Ok([])`. It has no encoding class of failure: any content is
  `Ok`. Its `Err` classes are `read`'s other three — the path is absent,
  names a directory, or cannot be read.
- `writeBytes` truncates first and creates when absent; `appendBytes` keeps
  what was there. Neither creates a missing parent, and both are `Err` on a
  path that names a directory. An empty `bytes` writes nothing, exactly as
  its text twin does with the empty string.
- `stringBytes` yields the bytes of a string, but the language has neither a
  bytes-to-string nor an integer-to-`u8` conversion: byte values come from
  `stringBytes`, from `readBytes`, or from literals, and arithmetic on them
  stays in `u8`. Bytes become text only when a text reader accepts them as
  valid UTF-8. `copy` moves a whole file without decoding; these three are
  for content the program inspects or produces itself.

```xulo
std::fs::writeBytes("./blob.bin", [0x7F, 0x45, 0x4C, 0x46])  // ok
std::fs::readBytes("./blob.bin")            // runs: Ok([127, 69, 76, 70])
std::fs::appendBytes("./blob.bin", [])      // ok — no bytes added
```

## Creating a file exclusively

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::createNew` | `fn(path: string, contents: string): Result<unit, Error>` | create a new file containing `contents`, failing if anything already exists at `path` |

- `createNew` is `write` with an existence check folded into the same
  operation: it succeeds **only** when `path` names nothing yet, and the check
  and the creation cannot come apart. A preceding `exists` call offers no such
  guarantee — another process can create the file in the gap between the two
  calls, so a check-then-write sequence decides on a fact that may already be
  stale.
- On success the file contains exactly `contents`, encoded as UTF-8 bytes;
  writing the empty string creates an empty file, which is success.
- Each of the following is `Err`:
  - the path already names a file — nothing is truncated, the file is left
    exactly as it was,
  - the path already names a directory,
  - the parent directory does not exist (`mkdir` makes directories), and
  - the path cannot be written (permissions, I/O error, path syntax the host
    rejects).
- The empty path names nothing and is `Err`.

```xulo
std::fs::createNew("./once.txt", "first")   // ok — the file did not exist
std::fs::createNew("./once.txt", "second")  // Err — it exists now
std::fs::exists("./once.txt")               // runs: true
// a read of "./once.txt" still yields Ok("first")
```

## Touching a file

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::touch` | `fn(path: string): Result<unit, Error>` | create an empty file when absent; refresh its modification time when present |

- `touch` never writes content: an existing file keeps every byte it held,
  and only its modification time changes — set to the current wall-clock
  time, the same reading [`std::time::now`](time.md) gives. It is `write`'s
  opposite in the one way that matters: `write` replaces content, `touch`
  replaces only the stamp.
- An absent path is created as an empty file, as an empty `write` would
  create it; the parent directory is **not** created — a missing parent is
  `Err`, as everywhere here.
- An existing directory is refreshed too: `touch` works on both kinds
  `stat` knows.
- On a path that exists, `touch` is
  [`setTimes`](#setting-the-modification-time) with the current time; what
  `touch` adds is creating the file when the path is absent.
- Each of the following is `Err`:
  - the path is empty,
  - the path is absent and its parent directory does not exist,
  - the path names something that is neither a file nor a directory (see
    [`stat`](#inspecting-a-file)), and
  - the path cannot be written (permissions, I/O error).

```xulo
std::fs::touch("./stamp.txt")   // ok — created, empty
std::fs::write("./stamp.txt", "x")
std::fs::touch("./stamp.txt")   // ok — still holds "x", the stamp is new
```

## Setting the modification time

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::setTimes` | `fn(path: string, modified: u64): Result<unit, Error>` | set the file's modification time to `modified` |

- `modified` is milliseconds since the Unix epoch — the same clock and unit
  as [`std::time::now`](time.md) and the `modified` field of
  [`stat`](#inspecting-a-file) — so a stamp read from `stat` can be written
  straight back, and `setTimes(path, std::time::now())` stamps the present.
- Content is never touched: only the stamp changes. The access time is not
  part of this module's surface — `stat` does not report it, so `setTimes`
  does not take it; when a host refreshes access time on its own, that is
  the host's business.
- Both kinds `stat` knows are eligible: files and directories.
- On a path that exists, this is the operation `touch` performs; what
  `touch` adds is creating the file when the path is absent. `setTimes`
  never creates.
- Each of the following is `Err`:
  - the path is empty,
  - the path does not exist (including a broken symlink),
  - the path names something that is neither a file nor a directory (see
    [`stat`](#inspecting-a-file)), and
  - the path cannot be written (permissions, I/O error).

```xulo
std::fs::setTimes("./note.txt", 0)            // ok — stamped at the epoch
match std::fs::stat("./note.txt") {
  Result::Ok(info) => print(info.modified)    // runs: 0
  Result::Err(e) => print(e.message)
}
```

## Setting the permission bits

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::setMode` | `fn(path: string, mode: number): Result<unit, Error>` | set the entry's permission bits to `mode` |

- `mode` is the low **12 bits** of a POSIX mode — the owner, group, and
  other read/write/execute bits, `0o777` when only the familiar nine
  matter. The domain is `0 …= 4095`; a fractional value counts as its
  truncation toward zero, and a value outside the domain is `Err`, the
  argument domain checked before the host is touched.
- Only the bits change: content, size, and the modification time stay as
  they were.
- The entry is found as `stat` finds it, so a link is followed to what it
  points at. A host without POSIX permission bits applies what it has and
  synthesizes `stat.mode` in return; the mapping is the host's (§ 5.5 of
  [`README.md`](README.md)).
- Each of the following is `Err`:
  - the path is empty,
  - the path does not exist (including a broken symlink),
  - the path names something `stat` does not know, and
  - the call is refused (permissions, I/O error).
- This is how a file this module wrote becomes runnable: `writeBytes` the
  image, then `setMode(path, 0o755)`.

```xulo
std::fs::setMode("./tool", 0o755)          // ok — readable and executable
std::fs::setMode("./tool", 4096)           // runs: Err — mode outside 0 …= 4095
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

## Resolving a path

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::realpath` | `fn(path: string): Result<string, Error>` | the absolute, link-free path naming what `path` names |

- `realpath` answers *where* a path leads: the result is absolute, spelled
  with the host's separator and case rules (§ 5.5 of
  [`README.md`](README.md)), with `.` and `..` components and duplicate
  separators gone and every symbolic link resolved. Naming the result again
  resolves to itself.
- Unlike `path::normalize`, which is pure string arithmetic on POSIX
  spellings ([`path.md`](path.md)), `realpath` asks the host: links are
  read, and the working directory supplies the base for a relative path.
- The path must name something: a missing component, a broken link, or a
  component that cannot be entered is `Err`. A link anywhere in the path —
  the last component included — is followed, so the result names the
  target, never the link.
- Spellings that differ only by `.`, `..`, duplicate separators, or links
  yield the same result, which is how a program deduplicates paths. A
  case-insensitive host may keep the case it stores, so string identity
  follows the host's rules, not a portable one.
- Each of the following is `Err`:
  - the path is empty,
  - a component does not exist (a broken link included), and
  - a component cannot be read (permissions, I/O error).
- `exists`, `realpath`, and `stat` divide the questions: `exists` says
  whether, `realpath` says where, and `stat` says what.

```xulo
std::fs::realpath("./src/../src/main.x")   // ok: an absolute path with no "." or ".."
std::fs::realpath("./nope")                // runs: Err — nothing to resolve
std::fs::realpath("./link-to-src")         // ok: the directory's own path — the link is gone
```

## Inspecting a file

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::stat` | `fn(path: string): Result<{ kind: string, size: number, modified: u64, mode: number }, Error>` | what the path is: its kind, its size, its modification time, and its permission bits |

- The result is a structural object with four fields:
  - `kind` — `"file"` or `"directory"`. Symlinks are followed, as `exists`
    follows them, so the answer names what the path resolves to, never
    `"symlink"`. Anything else a host may keep — a device, a socket, a
    broken link — is `Err`: this member knows two kinds.
  - `size` — the length of a file in bytes. For a directory the field is the
    host's own figure and carries no portable meaning.
  - `modified` — the last modification time in milliseconds since the Unix
    epoch, the same clock and unit as [`std::time::now`](time.md); a host
    whose clock reads before the epoch yields `0`, as `now` does.
  - `mode` — the permission bits, the low 12 bits of the host's POSIX mode,
    the value [`setMode`](#setting-the-permission-bits) writes. A host
    without POSIX permission bits synthesizes one (host-dependent, § 5.5 of
    [`README.md`](README.md)).
- Unlike `exists`, an unanswerable question is `Err` here: `stat` is asked
  for facts, and a probe that cannot be answered has no facts to give.
- Each of the following is `Err`:
  - the path does not exist (including a broken symlink),
  - the path names something that is neither a file nor a directory, and
  - the path cannot be read (permissions, I/O error).

The creation (birth) time is deliberately absent: many hosts and filesystems
record none, so it could only arrive as a nullable field — it stays out of
version 0.1 along with the other deferred `fs` members.

```xulo
match std::fs::stat("./note.txt") {
  Result::Ok(info) => print(info.kind == "file")   // runs: true
  Result::Err(e) => print(e.message)
}
```

## Copying a file

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::copy` | `fn(from: string, to: string): Result<unit, Error>` | copy `from`'s content to `to` |

- On success `to` holds exactly the bytes `from` held — a copy, so the
  original stays untouched.
- If `to` already exists it is **truncated and replaced**, as `write`
  replaces it; the previous content is gone. Its parent directory is **not**
  created: a missing parent is `Err`.
- Copying is not atomic: a copy interrupted partway can leave a partial `to`.
  A program that needs a whole-or-nothing result writes a temporary file in
  the destination's directory
  ([`tempFileIn`](#temporary-files-and-directories)) and `rename`s it into
  place: a temporary under the host's temporary directory can sit on another
  filesystem, where `rename` is `Err`.
- The copy is a new entry with the host's default permission bits: a copied
  executable is not executable until `setMode` says so, while `rename` keeps
  the bits because it is the same entry.
- Each of the following is `Err`:
  - `from` does not exist or names a directory,
  - `from` and `to` name the same path — a copy onto its source changes
    nothing and reports it,
  - `to`'s parent is missing, or `to` names an existing directory, and
  - either path cannot be read or written (permissions, I/O error).
- `copy` works at the byte level and never decodes: the source need not be
  valid UTF-8, so this member — unlike `read` — copies an image as happily as
  a text file.

```xulo
std::fs::copy("./note.txt", "./note.bak")   // ok — note.bak holds a copy
std::fs::copy("./a.txt", "./a.txt")         // Err — a copy onto its source
```

## Renaming a file or directory

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::rename` | `fn(from: string, to: string): Result<unit, Error>` | move `from` to `to`, replacing `to` when the two agree in kind |

- On success `from` no longer exists and `to` holds what `from` held: one
  path becomes the other, with no copy step in between. A directory moves
  as a whole — tree, files, and all — in this one call.
- A file at `to` is **replaced** — this is the "save" half of the
  write-temporarily-then-rename pattern a program uses when a reader must
  never observe half a file. A directory at `to` is replaced as well, but
  only when it is **empty**: `rename` never destroys the contents of the
  directory it overwrites. A non-empty `to` is `Err` — clearing it first is
  `removeDir`'s job, asked for plainly.
- The kinds agree or the call fails: a file cannot land where a directory
  stands, nor a directory where a file stands.
- Each of the following is `Err`:
  - `from` does not exist,
  - `from` and `to` name the same path — a move onto itself is no move,
  - the kinds disagree — a file onto a directory, or a directory onto a file,
  - `to` names a non-empty directory,
  - `to`'s parent directory does not exist, and
  - `from` and `to` lie on different filesystems — a move across mounts is
    not this member's job; a caller that needs one copies and removes.

```xulo
std::fs::rename("./note.tmp", "./note.txt")   // ok — note.txt is now the temp file
std::fs::rename("./build", "./build.old")     // ok — the whole tree moves in one call
std::fs::rename("./note.txt", "./note.txt")   // Err — no move
```

## Removing a file

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::remove` | `fn(path: string): Result<unit, Error>` | delete one file |

- `remove` deletes **one file**. A missing path is `Err`, and a second
  `remove` of the same path is `Err` as well — success means the file is gone.
- A directory path is `Err`: `remove` does not remove directories, empty or
  not; the member that deletes one — and its whole tree — is `removeDir`.
- The path names the entry itself: a symbolic link is removed as a link,
  and what it points at is left untouched. Permission checks follow the
  host's filesystem.

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

## Listing a directory

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::readDir` | `fn(path: string): Result<List<string>, Error>` | the names of everything in the directory at `path` |

- `readDir` yields one string per entry — files, directories, and whatever
  else the host keeps in a directory — as the **bare names**, not paths: the
  entries of `"./src"` are `"main.x"`, `"util.x"`, and so on. Joining one
  back onto `path` with [`std::path::join`](path.md) is the caller's step.
- The order of the names is the host's order: no sort order is promised, so a
  program that needs one sorts the list itself.
- An empty directory yields `Ok([])` — an empty list is content, not absence,
  mirroring `read`'s `Ok("")`.
- Each of the following is `Err`:
  - the path does not exist,
  - the path names a file rather than a directory, and
  - the directory cannot be read (permissions, I/O error).

```xulo
match std::fs::readDir("./src") {
  Result::Ok(names) => print(listSize(names))   // runs: 2
  Result::Err(e) => print(e.message)
}
```

## Walking a tree

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::walk` | `fn(path: string): Result<List<string>, Error>` | the starting path and every path beneath it, in one call |

- Each yielded string is the starting path extended downward — `walk("./src")`
  yields `"./src"`, `"./src/main.x"`, `"./src/lib/util.x"` — so every entry
  is a path `read` and `copy` accept as they are. The separator between
  levels is the host's, as the host spells every path here (§ 5.5 of
  [`README.md`](README.md)).
- The start itself is the first entry, and **a directory's own path always
  precedes every path beneath it**: a caller can reconstruct the tree in one
  forward pass. Beyond that guarantee no order is promised — the names come
  in the host's order, as with `readDir`.
- As in `readDir`, the paths are all the walk yields: kind and size come
  from [`stat`](#inspecting-a-file) when a caller needs them.
- The starting path is resolved as every member here resolves it — a link
  at the start is followed, so `walk` on a link to a directory walks that
  directory. A **symbolic link met on the way down** is never descended
  into: its path appears, but whatever it points at does not enter the
  walk. That keeps a link cycle from looping forever and keeps the result
  finite on every host.
- An empty directory yields `Ok([path])` — the start and nothing below it.
- The walk is all-or-nothing: each of the following is `Err`, and an `Err`
  yields no partial list —
  - the path does not exist (a broken link at the start does not resolve,
    so it fails here),
  - the path names a file rather than a directory, and
  - a directory on the way cannot be read (permissions, I/O error).
  A broken link *below* the start is not a failure: it is only a path the
  walk lists, and its `stat` is where the trouble will show.

```xulo
match std::fs::walk("./src") {
  Result::Ok(paths) => print(listSize(paths))   // runs: 3
  Result::Err(e) => print(e.message)
}
```

## Removing directories

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::removeDir` | `fn(path: string): Result<unit, Error>` | delete the directory at `path` and everything in it |

- `removeDir` is `mkdir`'s opposite: `mkdir` brings a whole chain into
  existence, `removeDir` takes the whole tree away — directories, files, and
  the nested directories between them. Nothing under `path` survives, and
  there is no undelete.
- Success means the path is gone: a second `removeDir` of the same path is
  `Err`, as a second `remove` is — absence is not an outcome of a removal
  call.
- A path that names a file is `Err`: `removeDir` removes directories; the
  file member is `remove`.
- The empty path names nothing and is `Err`.
- A symbolic link at `path` is not a directory and is `Err`: removal names
  the entry, not what it points at, and the target is never deleted. Remove
  the link itself with `remove`.
- A link met inside the tree is removed as a link, never descended into, so
  a link pointing outside the tree cannot widen the removal.
- Permission checks follow the host's filesystem.

```xulo
std::fs::mkdir("./scratch/tmp")        // ok — the chain exists
std::fs::removeDir("./scratch")        // ok — everything under it is gone
std::fs::exists("./scratch")           // runs: false
```

## Temporary files and directories

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::fs::tempFile` | `fn(prefix: string? = null): Result<string, Error>` | create a unique empty file in the host's temporary directory; yield its path |
| `std::fs::tempDir` | `fn(prefix: string? = null): Result<string, Error>` | create a unique empty directory in the host's temporary directory; yield its path |
| `std::fs::tempFileIn` | `fn(dir: string, prefix: string? = null): Result<string, Error>` | create a unique empty file in `dir`; yield its path |
| `std::fs::tempDirIn` | `fn(dir: string, prefix: string? = null): Result<string, Error>` | create a unique empty directory in `dir`; yield its path |

- The path is the product: `Ok` yields where a brand-new entry already
  lives. Which directory the host keeps temporaries in is host-dependent
  (§ 5.5 of [`README.md`](README.md)) — the member picks it, never the
  caller.
- The name always carries a part no other call predicts, so two calls never
  yield the same path. The entry is created exclusively — as `createNew`
  creates — before the path is yielded, so the caller's path names a real,
  empty, unshared entry from its first moment.
- `tempFile` yields a file that already exists and is empty: `read` on it is
  `Ok("")`, `write` replaces its content, `append` grows it. `tempDir`
  yields a directory that already exists and is empty: `readDir` on it is
  `Ok([])`.
- `prefix`, when given, heads the new entry's name; what completes the name
  is the host's business and no format is promised.
- Nothing expires or cleans itself up: removal belongs to the caller, as
  everywhere in this module — `remove` for the file, `removeDir` for the
  directory. A program that exits without cleaning up leaves the entry
  behind.
- `Err` when the host's temporary directory cannot be created in — absent,
  unwritable, no permission.
- The `…In` pair carries the same guarantees — unique, exclusively created,
  empty, never cleaned up — with the directory chosen by the caller instead
  of the host. `dir` must already exist and be a directory (`mkdir` makes
  it); a missing, unwritable, or non-directory `dir` is `Err`.
- Their reason to exist is the rename story: a temporary made beside its
  destination shares the destination's filesystem, so
  `rename` into place cannot cross a mount.

```xulo
match std::fs::tempFile("upload_") {
  Result::Ok(path) => print(path)                    // runs: a fresh path under the host's temp directory
  Result::Err(e) => print(e.message)
}
match std::fs::tempDir() {
  Result::Ok(dir) => print(std::fs::readDir(dir))    // runs: Ok([]) — it is empty
  Result::Err(e) => print(e.message)
}
```

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
| Fallible (`Result`) | `read` `readBytes` `write` `writeBytes` `append` `appendBytes` `createNew` `touch` `setTimes` `setMode` `realpath` `stat` `copy` `rename` `remove` `mkdir` `readDir` `walk` `removeDir` `tempFile` `tempFileIn` `tempDir` `tempDirIn` |
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
std::fs::append("a.txt", "hi")            // ok — a.txt now holds "hihi"
std::fs::createNew("b.txt", "hi")         // ok — "b.txt" did not exist
std::fs::touch("b.txt")                  // ok — content kept, stamp refreshed
std::fs::setTimes("b.txt", 0)            // ok — stamped at the Unix epoch
std::fs::realpath("./a.txt")              // ok: Result<string, Error> — absolute and link-free
std::fs::stat("a.txt")                    // ok: Result<{ kind, size, modified, mode }, Error>
std::fs::copy("a.txt", "c.txt")           // ok
std::fs::rename("c.txt", "d.txt")         // ok
std::fs::readDir(".")                     // ok: Result<List<string>, Error>
std::fs::walk(".")                        // ok: Result<List<string>, Error> — the start and everything below
std::fs::removeDir("./scratch")           // ok
std::fs::tempFile("upload_")              // ok: Result<string, Error> — a path that did not exist
std::fs::tempDir()                        // ok: Result<string, Error> — an empty directory
std::fs::tempFileIn("./build", "obj_")    // ok: Result<string, Error> — in ./build
std::fs::readBytes("a.txt")               // ok: Result<List<u8>, Error> — no decoding
std::fs::writeBytes("blob.bin", [0x7F, 0x45, 0x4C, 0x46])  // ok
std::fs::appendBytes("blob.bin", [])      // ok — no bytes to add
std::fs::setMode("blob.bin", 0o755)       // ok — executable bits on
std::fs::setMode("blob.bin", 4096)        // runs: Err — mode outside 0 …= 4095
std::fs::read(42)                         // error[E0202]: expected `string`, found `int`
std::fs::write("a.txt")                   // error[E0207]: expected 2 arguments, found 1
std::fs::scan(".")                        // error[E0601]: no member `scan`
Fs::read("a.txt")                         // error[E0607]: `Fs` lives under `std`: use `std::fs::read`
```
