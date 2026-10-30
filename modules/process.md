# `std::process`

`std::process` holds process-level control and child-process capture: five
functions — `exit` to terminate with a status, `args` to read the argument
vector, `run` and `exec` to execute a command and capture both of its
streams in one call, and `launch` to start one and keep a handle — plus two
types, `Command` (a child specification built in steps) and `Child` (a
running child). The module is host-dependent
([`README.md`](README.md) § 5.5): which shell `run` reaches, how `exec`
resolves a program, what the argument vector contains, and how an exit
status reaches the environment are the host's affair. The design keeps
failures as data ([`README.md`](README.md) § 5.1): a command that fails is
`Ok` with a non-zero `code`, and only four classes of problem reach `Err`.

The qualified path needs no import:

```xulo
let argv = std::process::args()   // ok: List<string>
std::process::exit(0)             // ok: terminates the program with status 0
let build = std::process::Command::direct("make", [])   // ok: a Command value
```

Import forms are the six of [`README.md`](README.md) § 2.2; the face is the
five functions and two types — no constants ([`README.md`](README.md) § 3):

```xulo
use std::process::*;              // exit args run exec launch Command Child
use std::process as proc;
proc::args()                      // ok: same member, aliased
use std::process::{Command, launch};
```

## Types

| Type | Shape | Role |
|------|-------|------|
| `Command` | static `shell(cmd)`, `direct(program, args)`; setters `input(s)`, `cwd(path)`, `env(name, value)`, `timeout(ms)`; methods `run`, `launch` | a child specification built in steps |
| `Child` | methods `wait`, `kill`, `pid` | a running child process |

Neither type has a public literal: a `Command` value comes only from its two
static constructors, and a `Child` only from a successful `launch`.

## Exit

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::process::exit` | `fn(code: number): never` | terminate this process with `code` as its exit status |

- The call returns nothing — it **never returns**. The checker types the
  call `null` so `std::process::exit(1)` is a legal expression statement
  (the statement-value rule polices discarded non-`unit` values,
  `spec/statements/expression-statements.md`); nothing after the call can
  run.
- Output is flushed before the process dies: the module pushes buffered
  stdout and stderr out, and an implementation that buffers program output
  of its own — the interpreter's collected prelude `print` lines — drains
  that buffer first, so output written before the call reaches the
  environment.
- `code` is taken as the process's exit status. The host truncates it by
  its own conventions — on Unix the low 8 bits, so `256 + n` exits with
  `n` — and the exit-status rules for ordinary termination live in
  `spec/memory-and-runtime.md`: a program that terminates normally exits
  with status **0**, one that ends in a runtime failure exits with a
  **nonzero** status, and the exact nonzero values are implementation-
  defined. `exit` is how a program chooses its status deliberately.

```xulo
std::process::exit(0)              // runs: terminates with status 0
std::process::exit(1)              // runs: terminates with status 1 — the final statement of a program
std::process::exit(256 + 1)        // runs: on Unix, terminates with status 1 — the host keeps the low 8 bits
```

## Arguments

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::process::args` | `fn(): List<string>` | the process's argument vector |

- The call is **total**: every process has an argument vector. Element 0 is
  the program path — the script or binary that started this process — and
  the remaining elements are the arguments the host passed.
- The language's entry point takes no parameters
  (`spec/modules/source-files.md`), so this vector is how a program reads
  its arguments.
- The vector's content is host-dependent ([`README.md`](README.md) § 5.5);
  the contract is the shape — a `List<string>` that starts with the program
  path. Nothing about a particular program's argv is portable.

```xulo
let argv = std::process::args()   // ok: List<string> — argv[0] is the program path
let n = listSize(argv)            // ok: at least 1; the exact vector is host-dependent
```

## Child capture

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::process::run` | `fn(cmd: string, input: string? = null, timeout: u64? = null): Result<{ stdout: string, stderr: string, code: number }, Error>` | run `cmd` through the shell and capture both streams |
| `std::process::exec` | `fn(program: string, args: List<string>, input: string? = null, timeout: u64? = null): Result<{ stdout: string, stderr: string, code: number }, Error>` | run `program` directly with `args` and capture both streams |

**The shell form.** `run`'s `cmd` runs through the **POSIX shell** (`sh -c`),
so shell syntax — pipelines, redirections, variable expansion — is part of
the command's meaning. A host where the shell cannot be started yields a
clean `Err`, never a crash.

**The direct form.** `exec` starts `program` without a shell: `args` become
the child's argument vector exactly as given — no metacharacters, no quoting
to get right, and a `"; reboot"` inside one argument is that argument, not a
command. The host resolves `program` the way it resolves programs (its
search path); one that is not found, not executable, or refused by the host
yields the same clean `Err` the shell form uses for an unstartable shell.
What `exec` does not have is the shell: a pipeline or a redirection is a
`run`.

**Both forms.**

- The trailing arguments are optional: `run` takes one to three positional
  arguments, `exec` two to four, and a call past those counts is `E0207`
  ([`README.md`](README.md) § 4 lists every member with an optional
  trailing argument and its range). An omitted `input` is `null`; an
  omitted `timeout` is `null`.
- The call **blocks** until the child finishes — a blocking member in the
  sense of [`README.md`](README.md) § 5.6: it freezes the cooperative
  scheduler for the duration, so nothing else makes progress while the
  child runs. Version 0.1 has no asynchronous variant; that is a roadmap
  candidate (`std::async`, [`../README.md`](../README.md) § 1.2). A
  `timeout` is how a program keeps that freeze bounded.
- **timeout** is a budget in milliseconds, measured from the moment the
  child starts. `null` waits without limit — the behavior before `timeout`
  existed, kept as the default. A budget that expires has the host kill the
  child by forceful means and yields `Err`: a third class beside the cap,
  because the program's own budget broke, not the child's behavior.
  Whatever the two streams had produced by then is discarded, exactly as an
  over-cap capture discards its stream. A child that finishes inside the
  budget never reports a timeout, however close the finish.
- **stdin.** With `input` a string, the child's stdin is that string: the
  stream is piped, the string is written, and the pipe is closed so
  `cat`-style children see end of input. With `input` omitted — or passed
  as `null` — the child **inherits this process's stdin**; the omitted
  form does not mean an empty stdin.
- **stdout and stderr** are always captured, both decoded lossily into
  strings: bytes that are not valid UTF-8 become U+FFFD replacement
  characters — the same policy [`net.md`](net.md)'s `sockRead` shares. The
  child inherits this process's environment, so variables set through
  `std::os::setenv` are visible to it ([`os.md`](os.md)).
- **`code`** is the child's exit status; `-1` when a signal killed the
  child, which has no status.
- **A non-zero exit code is data, not an error.** The call yields `Ok` with
  `code != 0` — the shape the caller matches — and never reports a failing
  command as `Err`.
- The call yields `Err` for exactly three classes of problem: the child
  cannot be started (the shell for `run`, the program for `exec`), a
  captured stream crosses the capture cap, and `timeout` expires. The cap
  is per stream: **more than 16777216 bytes (16 MiB) on either stream
  fails the whole call**, while a stream of exactly 16777216 bytes is
  captured whole. Both streams are drained concurrently, and a reader that
  crosses the cap kills the child before the error is reported, so a full
  pipe cannot wedge the call. The `message` of the `Error` names the
  command and the OS reason (informative, [`../README.md`](../README.md)
  § 3.5); the contract is the `Err` shape and the three classes above.
- A program that wants a child's stderr on its own stderr writes it with
  `std::io::eprintln` ([`io.md`](io.md)).
- These two members are the one-shot form: start, capture, done. A child
  that must be held — to give it a working directory or environment, to
  kill it, or to wait for it later — is [`Command`](#command) and
  [`launch`](#children) below. Version 0.1 has no live, two-way stdin
  either; that is a roadmap candidate too.

```xulo
let hello = std::process::run("echo hello")
// runs: Ok({ stdout: "hello\n", stderr: "", code: 0 })

let fail = std::process::run("exit 3")
// runs: Ok with code 3 — a non-zero exit is data, not an error

let up = std::process::run("cat", "text")
// runs: Ok with stdout "text" — the two-argument form pipes the input

let listed = std::process::exec("ls", ["-a"])
// runs: Ok with the directory listing — one argument, nothing to quote

let timed = std::process::run("sleep 10", null, 500)
// runs: Err — the 500 ms budget expires and the child is killed

std::process::run("true")
// runs: Ok with code 0 — input omitted: stdin is inherited, not emptied

match std::process::run("echo oops 1>&2; exit 3") {
  Result::Ok(out) => std::io::eprintln(out.stderr)  // runs: "oops" on our stderr; out.code is 3
  Result::Err(e) => std::io::eprintln(e.message)    // a start failure, an oversized capture, or a timeout
}
```

## Command

A `Command` is a child specification built in steps: the two static
constructors pick the shape — the same shell and direct forms `run` and
`exec` name — and the setters refine it. `run` executes it to completion;
`launch` starts it behind a `Child` handle.

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Command::shell` | `fn(cmd: string): Command` | a command that runs `cmd` through the POSIX shell, like `run` |
| `Command::direct` | `fn(program: string, args: List<string>): Command` | a command that runs `program` with `args`, no shell, like `exec` |
| `Command::input` | `fn(s: string): Command` | pipe `s` to the child's stdin and close it, like `run`'s `input` |
| `Command::cwd` | `fn(path: string): Command` | start the child in `path` instead of this process's directory |
| `Command::env` | `fn(name: string, value: string): Command` | overlay one variable on the inherited environment |
| `Command::timeout` | `fn(ms: u64): Command` | budget the child's lifetime, as `run`'s `timeout` does |
| `Command::run` | `fn(): Result<{ stdout: string, stderr: string, code: number }, Error>` | run to completion and capture, exactly as `run`/`exec` |
| `Command::launch` | `fn(): Result<Child, Error>` | start the child and return a handle |

- **Setters are pure.** Each returns a `Command` with that one setting
  applied; the receiver is unchanged, so settings compose in any order.
- **One value, repeated calls.** `run` and `launch` may be called on the
  same `Command` repeatedly, each call starting a fresh child with the
  same settings.
- **`cwd` is not checked when set**, only when used: a directory that does
  not exist, or that is not enterable, fails the call in the start class —
  the child never starts — and the error names the command.
- **`env` is an overlay, not a replacement.** The child inherits this
  process's environment and the overlay wins on a collision. Repeated
  `env` calls compose; the later one wins.
- **The shorthands are these commands.** `run(cmd)` is
  `Command::shell(cmd).run()`, `exec(program, args)` is
  `Command::direct(program, args).run()`, and `launch(cmd)` is
  `Command::shell(cmd).launch()` — each with no other setting.
- The `timeout` budget measures the child's lifetime from the moment the
  child starts, whether the call is `run` (one shot) or `launch` (observed
  when `wait` runs; see below).

```xulo
let make = std::process::Command::shell("make")           // ok: a Command value
let build = make.cwd("build").env("CC", "clang").timeout(60000)
let done = build.run()      // ok: Result<{ stdout, stderr, code }, Error> — 60 s for the build
```

## Children

`launch` starts a child and returns a `Child` handle: the program keeps
running while this process does other work, and the handle is how the
program later collects it or stops it.

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::process::launch` | `fn(cmd: string, input: string? = null, timeout: u64? = null): Result<Child, Error>` | start `cmd` through the shell, like `run` without the wait |
| `Child::wait` | `fn(): Result<{ stdout: string, stderr: string, code: number }, Error>` | block for the child, capture both streams, and reap it |
| `Child::kill` | `fn(): Result<unit, Error>` | terminate the child forcefully |
| `Child::pid` | `fn(): number` | the child's process id |

- The member is `launch`, not `spawn`: `spawn` is a keyword of the
  language — `spawn async { … }`, `spawn.process async { … }` — so no path
  can name a member `spawn`. The language's `spawn` starts Xulo work, a
  block or a service; `launch` starts an external program. The two never
  compete for a name.
- `launch` is the shell-form shorthand; `Command::launch` is the full form
  and is the only way to start a child in its own directory or with an
  environment overlay.
- **`wait` is `run` delayed.** It blocks, drains both streams while it
  waits — so a chatty child cannot wedge on a full pipe — applies the same
  16 MiB cap and lossy UTF-8 decoding, and yields the same record. A
  non-zero `code` is still data.
- **The `timeout` rides on the child.** A budget from `launch` (or
  `Command::timeout` before `Command::launch`) bounds the child's lifetime
  from its start. Version 0.1 has no background watcher, so the expiry is
  observed when `wait` runs: a child that outlived its budget is killed
  then, and `wait` yields `Err`. A child that finished inside the budget
  yields its record even if `wait` is called later.
- **`kill` delivers a forceful termination** and yields `Ok` once it is
  delivered. It yields `Err` when there is nothing to kill — the child has
  already exited, or the handle is spent. Killing is not reaping: the
  child's status is still collected by `wait`, whose `code` is `-1` when
  the kill was what ended it.
- **A handle is spent once `wait` returns**, with `Ok` or `Err` alike:
  every later `wait` or `kill` on it yields `Err`. `pid` keeps answering —
  the id it was given is a fact.
- **stdin is fixed at launch.** A `Command::input` string is piped and
  closed before the child runs; an omitted input inherits this process's
  stdin. Version 0.1 has no live stdin: writing to a running child is a
  roadmap candidate.

```xulo
let running = std::process::launch("long-task")
// ok: Result<Child, Error> — the child starts now

let bounded = std::process::Command::shell("long-task").timeout(5000).launch()
// ok: Result<Child, Error> — a 5 s lifetime budget, observed by wait

match std::process::launch("slow-server") {
  Result::Ok(child) => std::io::println(str(child.pid()))   // runs: the child's pid
  Result::Err(e) => std::io::eprintln(e.message)
}
```

## Failure model

| Kind | Members |
|------|---------|
| Never returns | `exit` — it terminates the process, so there is no return to classify |
| Total | `args`; `Command::shell`, `Command::direct`, the setters `input`, `cwd`, `env`, `timeout`; `Child::pid` |
| Fallible (`Result`) | `run`, `exec`, `launch`, `Command::run`, `Command::launch`, `Child::wait`, `Child::kill` — four classes: the child cannot start; a captured stream crosses the cap; `timeout` expires; the handle cannot serve the call |
| Absence (`null`) | none |
| Runtime failure | none — no `std::process` member has a runtime-failure argument domain |

`exit` is neither fallible nor an ordinary total member: it never returns,
and its checker type `null` exists only so the call composes in statement
position. A non-zero child exit code is the `code` field of `Ok`, never an
`Err`. The handle class belongs to `Child` alone: a second `wait`, or a
`kill` of a child that has already exited or of a spent handle, is a
refusal to answer twice, not a child failure.

## Checked call examples

```xulo
std::process::args()                 // ok: List<string>
std::process::run("echo hello")      // ok: Result<{ stdout, stderr, code }, Error>
std::process::exec("ls", ["-a"])     // ok: Result<{ stdout, stderr, code }, Error>
std::process::launch("make")         // ok: Result<Child, Error>
std::process::Command::shell("make").run()   // ok: Result<{ stdout, stderr, code }, Error>
std::process::Command::direct("ls", ["-a"]).timeout(500).run()   // ok
std::process::exit(1)                // ok: typed null — a legal expression statement
std::process::run()                  // error[E0207]: expected 1 to 3 arguments, found 0
std::process::run("x", "y", "z", "w")   // error[E0207]: expected 1 to 3 arguments, found 4
std::process::exec("ls")             // error[E0207]: expected 2 to 4 arguments, found 1
std::process::launch()               // error[E0207]: expected 1 to 3 arguments, found 0
std::process::run(1, "x")            // error[E0202]: expected `string`, found `int`
std::process::exec("ls", "-a")       // error[E0202]: expected `List<string>`, found `string`
std::process::signal("TERM")         // error[E0601]: no member `signal`
Process::exit(1)                     // error[E0607]: `Process` lives under `std`: use `std::process::exit`
```
