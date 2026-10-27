# `std::os`

`std::os` holds the small surface a program needs to talk to the operating
system it runs on: the environment, the current working directory, well-known
host directories, and process and host identity. Its fallible members report
failure in the `Result` slot; `pid`, `platform`, `arch`, `tmpDir`, and
`cpuCount` are total and never fail. The module is host-dependent — which
variables exist, which directory is current or temporary, and which strings
`platform`/`arch` yield depend on the host
([`README.md`](README.md) § 5.5).

The module's face is fourteen functions and nothing else: no constants and
no types ([`README.md`](README.md) § 3). The qualified path needs no import:

```xulo
std::os::cwd()                    // ok: Result<string, Error>
print(std::os::platform())        // runs: e.g. "linux" on this host
```

Import forms are the six of [`README.md`](README.md) § 2.2:

```xulo
use std::os::*;                   // the fourteen functions
use std::os as os;
os::pid()                         // ok: same member, aliased
use std::os::{getenv, setenv};
```

## Environment

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::os::getenv` | `fn(name: string): Result<string?, Error>` | the value of an environment variable |
| `std::os::setenv` | `fn(name: string, value: string): Result<unit, Error>` | set a variable for this process |
| `std::os::unsetenv` | `fn(name: string): Result<unit, Error>` | remove a variable from this process's environment |

- `getenv` distinguishes three cases. A variable that is set yields
  `Ok(value)`. A variable that is unset yields `Ok(null)` — **absence**, the
  same optional-string shape [`io.md`](io.md)'s `readLine` uses for
  end-of-stream. A variable that is set but whose value is not valid UTF-8
  yields `Err`: that is a failure, not a lie of `null` — absence means
  *unset*, failure means *unreadable*. A variable set to the empty string is
  `Ok("")`, distinguishable from unset.
- `setenv` fails with `Err` for an empty name, a name containing `=`, and a
  name or value containing a NUL byte; any other name/value pair is set for
  this process and the call yields `Ok`. A rejected `setenv` changes
  nothing: a subsequent `getenv` of the same name sees the prior state.
- `unsetenv` removes the variable: a later `getenv` of the name is
  `Ok(null)`, and a child started afterwards does not see it. Removing a
  name that is not set is `Ok` and changes nothing — removal is idempotent.
  It fails with `Err` for the three name classes `setenv` also rejects —
  empty, containing `=`, containing a NUL byte — and for nothing else.
- A set or a removal is process-wide: a later `getenv` of the same name
  observes it, and so does a child process spawned afterwards
  ([`process.md`](process.md)). Xulo tasks are cooperative on one OS thread,
  so no concurrent reader observes a torn environment.
- A test that asserts an environment value must set it first: an unset
  variable reads back as `Ok(null)`, never as a recorded empty string.

```xulo
let missing = std::os::getenv("XULO_NO_SUCH_VARIABLE")
// runs: Result::Ok(null) — unset is absence, not failure

match std::os::setenv("XULO_GREETING", "hello") {
  Result::Ok(_) => null
  Result::Err(e) => std::io::eprintln(e.message)
}
match std::os::getenv("XULO_GREETING") {
  Result::Ok(v) => std::io::println(str(v))   // runs: hello
  Result::Err(e) => std::io::eprintln(e.message)
}
```

## Working directory

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::os::cwd` | `fn(): Result<string, Error>` | the process's current working directory |
| `std::os::chdir` | `fn(path: string): Result<unit, Error>` | change the process's current working directory |

- `cwd` yields the current working directory as a string. A directory that
  has since been deleted, or any other host error, yields `Err`.
- `chdir` yields `Err` for a path that does not exist (or is otherwise not
  enterable); a path it accepts becomes the process's current directory.
- Subsequent operations that resolve relative paths — [`fs.md`](fs.md)
  members, and comparisons against a later `cwd()` — observe the new
  directory.

```xulo
match std::os::cwd() {
  Result::Ok(dir) => std::io::println(dir)    // runs: an absolute path string
  Result::Err(e) => std::io::eprintln(e.message)
}

match std::os::chdir("/tmp") {
  Result::Ok(_) => null
  Result::Err(e) => std::io::eprintln(e.message)
}
match std::os::cwd() {
  Result::Ok(dir) => std::io::println(dir)    // runs: now /tmp
  Result::Err(e) => std::io::eprintln(e.message)
}
```

## Host directories

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::os::tmpDir` | `fn(): string` | the host's temporary directory |
| `std::os::homeDir` | `fn(): Result<string, Error>` | the current user's home directory |

- `tmpDir` yields the value of the host's temporary-directory variable
  (`TMPDIR` on a POSIX host, the host's own where it differs) when it is set
  and non-empty, else the platform default — `/tmp` on a POSIX host. It is
  total and never checks that the directory exists: probing and creating are
  [`fs.md`](fs.md)'s work (`tempFile` and `tempDir` already create there).
- `homeDir` yields the home directory from the variable the host defines —
  `HOME` on a POSIX host, the host's own where it differs. A variable that is
  unset or set to the empty string yields `Err`: no home is discoverable,
  which is a failure to answer, not an absence — `homeDir` has no `Ok(null)`.
  Any other host error yields `Err` as well.

```xulo
print(std::os::tmpDir())             // runs: e.g. /tmp

match std::os::homeDir() {
  Result::Ok(dir) => std::io::println(dir)   // runs: an absolute path string
  Result::Err(e) => std::io::eprintln(e.message)
}
```

## Process and host identity

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::os::pid` | `fn(): number` | the process id |
| `std::os::platform` | `fn(): string` | the operating-system family |
| `std::os::arch` | `fn(): string` | the CPU architecture |
| `std::os::hostName` | `fn(): Result<string, Error>` | the machine's name |
| `std::os::userName` | `fn(): Result<string, Error>` | the user the process runs as |
| `std::os::execPath` | `fn(): Result<string, Error>` | the executable the host started for this process |
| `std::os::cpuCount` | `fn(): number` | the logical processors available to the process |

- `pid`, `platform`, `arch`, and `cpuCount` are total: every process has an
  id, a running program is always on one OS family and one CPU architecture,
  and a host always presents at least one execution context. They never
  return `Result` and never `null`, and `cpuCount` is never `0`.
- `pid`, `platform`, and `arch` are drawn from the host and do not change
  during the process's lifetime. `hostName` and `userName` are `Err` when
  the host will not say — no name configured, no account behind the process,
  or the reading refused.
- `platform` is the host's OS-family constant — `"linux"`, `"macos"`,
  `"windows"`, and whatever else the host defines. The list is open: a
  program treats a value it does not recognize as an opaque string.
- `arch` is the host's CPU-architecture constant — `"x86_64"`, `"aarch64"`,
  and whatever else the host defines — with the same open-list rule.
- `hostName` and `userName` are opaque strings in the same sense: a program
  compares or prints them, never parses them. `cpuCount` is a capacity, not
  a reservation — a program sizes work with it and does not assume the
  processors stay idle for it.
- `execPath` names where the program lives: an absolute path with symbolic
  links resolved, as [`std::fs::realpath`](fs.md#resolving-a-path) resolves
  them, so it names the executable file itself — for a compiled program the
  program, for one run by an interpreter the interpreter. It is how a tool
  finds files shipped beside itself — its standard library, resources —
  where `args()[0]` only repeats what the caller typed. A host that will
  not report the executable yields `Err`; the answer is fixed at process
  start.

```xulo
print(std::os::pid())              // runs: the process id, e.g. 12345
print(std::os::platform())         // runs: "linux" on this host
print(std::os::arch())             // runs: "x86_64" on this host
print(std::os::cpuCount())         // runs: e.g. 8 — at least 1 on any host

match std::os::hostName() {
  Result::Ok(name) => std::io::println(name)   // runs: the machine's name
  Result::Err(e) => std::io::eprintln(e.message)
}
```

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | `getenv` (value not valid UTF-8); `setenv` (empty name, `=` in name, NUL in name or value); `unsetenv` (empty name, `=` in name, NUL in name); `cwd` (host error); `chdir` (path not enterable); `homeDir` (variable unset, empty, or unreadable); `hostName`, `userName` (host error); `execPath` (host will not report the executable) |
| Total | `pid`, `platform`, `arch`, `tmpDir`, `cpuCount` |
| Absence | `getenv` → `Ok(null)` when the variable is unset — the `std::os` row of [`README.md`](README.md) § 5.2 |
| Runtime failure | none |

No `std::os` member stops the program on the program's own inputs: the nine
fallible members answer every bad name, unreadable value, and unenterable
path with `Err`, and the five total ones answer every call.

## Checked call examples

```xulo
std::os::getenv("HOME")           // ok: Result<string?, Error>
std::os::setenv("A", "1")         // ok
std::os::unsetenv("A")            // ok — and now `getenv("A")` is Ok(null)
std::os::pid()                    // ok: number
std::os::platform()               // ok: string
std::os::chdir("/tmp")            // ok
std::os::tmpDir()                 // ok: string
std::os::homeDir()                // ok: Result<string, Error>
std::os::execPath()               // ok: Result<string, Error> — where this program lives
std::os::cpuCount()               // ok: number — at least 1
std::os::getenv()                 // error[E0207]: expected 1 argument, found 0
std::os::setenv(1, "x")           // error[E0202]: expected `string`, found `int`
std::os::chdir(true)              // error[E0202]: expected `string`, found `bool`
std::os::arch("x")                // error[E0207]: expected 0 arguments, found 1
std::os::execPath(0)              // error[E0207]: expected 0 arguments, found 1
Os::cwd()                         // error[E0607]: `Os` lives under `std`: use `std::os::cwd`
std::os::environ()                // error[E0601]: no member `environ`
```
