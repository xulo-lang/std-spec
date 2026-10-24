# `std::os`

`std::os` holds the small surface a program needs to talk to the operating
system it runs on: the environment, the current working directory, and
process and host identity. Its fallible members report failure in the
`Result` slot; `pid`, `platform`, and `arch` are total and never fail. The
module is host-dependent — which variables exist, which directory is
current, and which strings `platform`/`arch` yield depend on the host
([`README.md`](README.md) § 5.5).

The module's face is seven functions and nothing else: no constants and no
types ([`README.md`](README.md) § 3). The qualified path needs no import:

```xulo
std::os::cwd()                    // ok: Result<string, Error>
print(std::os::platform())        // runs: e.g. "linux" on this host
```

Import forms are the six of [`README.md`](README.md) § 2.2:

```xulo
use std::os::*;                   // the seven functions
use std::os as os;
os::pid()                         // ok: same member, aliased
use std::os::{getenv, setenv};
```

## Environment

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::os::getenv` | `fn(name: string): Result<string?, Error>` | the value of an environment variable |
| `std::os::setenv` | `fn(name: string, value: string): Result<unit, Error>` | set a variable for this process |

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
- The change is process-wide: a later `getenv` of the same name observes it,
  and so does a child process spawned afterwards
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

## Process and host identity

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::os::pid` | `fn(): number` | the process id |
| `std::os::platform` | `fn(): string` | the operating-system family |
| `std::os::arch` | `fn(): string` | the CPU architecture |

- All three are total: every process has an id, and a running program is
  always on one OS family and one CPU architecture. They never return
  `Result` and never `null`.
- The three values are drawn from the host and do not change during the
  process's lifetime.
- `platform` is the host's OS-family constant — `"linux"`, `"macos"`,
  `"windows"`, and whatever else the host defines. The list is open: a
  program treats a value it does not recognize as an opaque string.
- `arch` is the host's CPU-architecture constant — `"x86_64"`, `"aarch64"`,
  and whatever else the host defines — with the same open-list rule.

```xulo
print(std::os::pid())              // runs: the process id, e.g. 12345
print(std::os::platform())         // runs: "linux" on this host
print(std::os::arch())             // runs: "x86_64" on this host
```

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | `getenv` (value not valid UTF-8); `setenv` (empty name, `=` in name, NUL in name or value); `cwd` (host error); `chdir` (path not enterable) |
| Total | `pid`, `platform`, `arch` |
| Absence | `getenv` → `Ok(null)` when the variable is unset — the `std::os` row of [`README.md`](README.md) § 5.2 |
| Runtime failure | none |

No `std::os` member stops the program on the program's own inputs: the four
fallible members answer every bad name, unreadable value, and unenterable
path with `Err`, and the three total ones answer every call.

## Checked call examples

```xulo
std::os::getenv("HOME")           // ok: Result<string?, Error>
std::os::setenv("A", "1")         // ok
std::os::pid()                    // ok: number
std::os::platform()               // ok: string
std::os::chdir("/tmp")            // ok
std::os::getenv()                 // error[E0207]: expected 1 argument, found 0
std::os::setenv(1, "x")           // error[E0202]: expected `string`, found `int`
std::os::chdir(true)              // error[E0202]: expected `string`, found `bool`
std::os::arch("x")                // error[E0207]: expected 0 arguments, found 1
Os::cwd()                         // error[E0607]: `Os` lives under `std`: use `std::os::cwd`
std::os::environ()                // error[E0601]: no member `environ`
```
