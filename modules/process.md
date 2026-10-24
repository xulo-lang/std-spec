# `std::process`

`std::process` holds process-level control and child-process capture: three
functions — `exit` to terminate with a status, `args` to read the argument
vector, and `run` to execute a command and capture both of its streams. The
module is host-dependent ([`README.md`](README.md) § 5.5): which shell `run`
reaches, what the argument vector contains, and how an exit status reaches
the environment are the host's affair. The design keeps failures as data
([`README.md`](README.md) § 5.1): a command that fails is `Ok` with a
non-zero `code`, and only two classes of problem reach `Err`.

The qualified path needs no import:

```xulo
let argv = std::process::args()   // ok: List<string>
std::process::exit(0)             // ok: terminates the program with status 0
```

Import forms are the six of [`README.md`](README.md) § 2.2; the face is the
three functions — no constants and no types ([`README.md`](README.md) § 3):

```xulo
use std::process::*;              // exit args run
use std::process as proc;
proc::args()                      // ok: same member, aliased
use std::process::{run};
```

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
let n = argv.length                // ok: at least 1; the exact vector is host-dependent
```

## Child capture

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::process::run` | `fn(cmd: string, input: string? = null): Result<{ stdout: string, stderr: string, code: number }, Error>` | run `cmd` to completion and capture both streams |

- `cmd` runs through the **POSIX shell** (`sh -c`), so shell syntax —
  pipelines, redirections, variable expansion — is part of the command's
  meaning. A host where the shell cannot be started yields a clean `Err`,
  never a crash.
- The second argument is optional: one or two positional arguments are
  accepted, and a third is `E0207` ([`README.md`](README.md) § 4 names
  `run` as one of the two members that take an optional trailing argument;
  the omitted argument takes the default stated here). An omitted `input`
  is `null`.
- The call **blocks** until the child finishes — a blocking member in the
  sense of [`README.md`](README.md) § 5.6: it freezes the cooperative
  scheduler for the duration, so nothing else makes progress while the
  child runs. Version 0.1 has no asynchronous variant; that is a roadmap
  candidate (`std::async`, [`../README.md`](../README.md) § 1.2).
- **stdin.** With `input` a string, the child's stdin is that string: the
  stream is piped, the string is written, and the pipe is closed so
  `cat`-style children see end of input. With `input` omitted — or passed
  as `null` — the child **inherits this process's stdin**; the omitted
  form does not mean an empty stdin.
- **stdout and stderr** are always captured, both decoded lossily into
  strings: bytes that are not valid UTF-8 become U+FFFD replacement
  characters — the same policy [`net.md`](net.md)'s `sock_read` shares. The
  child inherits this process's environment, so variables set through
  `std::os::setenv` are visible to it ([`os.md`](os.md)).
- **`code`** is the child's exit status; `-1` when a signal killed the
  child, which has no status.
- **A non-zero exit code is data, not an error.** The call yields `Ok` with
  `code != 0` — the shape the caller matches — and never reports a failing
  command as `Err`.
- The call yields `Err` for exactly two classes of problem: the shell
  cannot be started, and a captured stream crosses the capture cap. The
  cap is per stream: **more than 16777216 bytes (16 MiB) on either stream
  fails the whole call**, while a stream of exactly 16777216 bytes is
  captured whole. Both streams are drained concurrently, and a reader that
  crosses the cap kills the child before the error is reported, so a full
  pipe cannot wedge the call. The `message` of the `Error` names the
  command and the OS reason (informative, [`../README.md`](../README.md)
  § 3.5); the contract is the `Err` shape and the two classes above.
- A program that wants a child's stderr on its own stderr writes it with
  `std::io::eprintln` ([`io.md`](io.md)).

```xulo
let hello = std::process::run("echo hello")
// runs: Ok({ stdout: "hello\n", stderr: "", code: 0 })

let fail = std::process::run("exit 3")
// runs: Ok with code 3 — a non-zero exit is data, not an error

let up = std::process::run("cat", "text")
// runs: Ok with stdout "text" — the two-argument form pipes the input

std::process::run("true")
// runs: Ok with code 0 — input omitted: stdin is inherited, not emptied

match std::process::run("echo oops 1>&2; exit 3") {
  Result::Ok(out) => std::io::eprintln(out.stderr)  // runs: "oops" on our stderr; out.code is 3
  Result::Err(e) => std::io::eprintln(e.message)    // only a spawn failure or an oversized capture
}
```

## Failure model

| Kind | Members |
|------|---------|
| Never returns | `exit` — it terminates the process, so there is no return to classify |
| Total | `args` |
| Fallible (`Result`) | `run` — the shell cannot start; a captured stream crosses the cap |
| Absence (`null`) | none |
| Runtime failure | none — no `std::process` member has a runtime-failure argument domain |

`exit` is neither fallible nor an ordinary total member: it never returns,
and its checker type `null` exists only so the call composes in statement
position. A non-zero child exit code is the `code` field of `Ok`, never an
`Err`.

## Checked call examples

```xulo
std::process::args()                 // ok: List<string>
std::process::run("echo hello")      // ok: Result<{ stdout, stderr, code }, Error>
std::process::exit(1)                // ok: typed null — a legal expression statement
std::process::run()                  // error[E0207]: expected 1 or 2 arguments, found 0
std::process::run("x", "y", "z")     // error[E0207]: expected 1 or 2 arguments, found 3
std::process::run(1, "x")            // error[E0202]: expected `string`, found `int`
std::process::spawn("x")             // error[E0601]: no member `spawn`
Process::exit(1)                     // error[E0607]: `Process` lives under `std`: use `std::process::exit`
```
