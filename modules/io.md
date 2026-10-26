# `std::io`

`std::io` holds the process's standard streams — stdout, stderr, and stdin —
as the real, running operations a program talks to: every write flushes
before it returns, and the reads drain stdin. Every member but the two
probes is fallible: it returns a `Result`, and a closed or broken stream
arrives in the `Err` slot as data, never as a panic
([`README.md`](README.md) § 5.1). The two terminal probes are total. The
module is host-dependent ([`README.md`](README.md) § 5.5): which bytes the
streams carry is the process environment's affair.

The qualified path needs no import:

```xulo
std::io::print("io: ")                 // ok: stdout, no terminator, flushed
std::io::println("hello stdout")       // ok: a newline appended, flushed
std::io::eprintln("io: hello stderr")  // ok: the same write, on stderr
```

Import forms are the six of [`README.md`](README.md) § 2.2, with this
module's one deviation: the face is eight of the ten exported names —
`eprint` `eprintln` `flush` `readLine` `read` `readN` `stdinIsTTY`
`stdoutIsTTY` — and `print`/`println` are
withheld from `use` because their local names are prelude intrinsics
(`E0601`). The rule and its reason live in [`README.md`](README.md) § 3
(with § 2.3's rejection table); what matters here is that the two members
behind it are the same flushing writes:

```xulo
use std::io as io;
io::print("prompt: ")            // ok: exactly std::io::print — flushed before it returns
use std::io::*;                  // binds the face: eprint eprintln flush readLine read readN stdinIsTTY stdoutIsTTY
// `use std::io::print;` and `use std::io as print;` are error[E0601]
```

## Writes

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::io::print` | `fn(text: string): Result<unit, Error>` | write `text` to stdout with no line terminator |
| `std::io::println` | `fn(text: string): Result<unit, Error>` | write `text` and a `\n` to stdout |
| `std::io::eprint` | `fn(text: string): Result<unit, Error>` | write `text` to stderr with no line terminator |
| `std::io::eprintln` | `fn(text: string): Result<unit, Error>` | write `text` and a `\n` to stderr |

- The text is written as the `string` it is — the bytes of `text` and, for
  the `println` pair, one `\n` — with no spacing, no rendering of other
  values, and nothing implicit; interpolate or `str` a value first.
- **Every write flushes its stream before the call returns.** Output cannot
  sit in a buffer while the program waits: a prompt written with `print` is
  visible before a later `readLine()` blocks. This is the contract of these
  members. The language's bare prelude `print` is a different path with
  different buffering (the interpreter collects it until the program ends);
  for that intrinsic see `spec/builtins/prelude.md`.
- Success is `Ok`; an `Err` means the write or the flush failed — a closed
  or broken stream, a pipe with no reader. The `message` of the `Error`
  names the stream and the OS reason (informative,
  [`../README.md`](../README.md) § 3.5); the contract is the `Err` shape
  and this failure class.

```xulo
std::io::print("io: ")                 // runs: "io: " on stdout, already flushed
std::io::println("hello stdout")       // runs: "io: hello stdout" + "\n"
std::io::eprintln("io: hello stderr")  // runs: "io: hello stderr" + "\n" on stderr
```

## Flush

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::io::flush` | `fn(): Result<unit, Error>` | push anything buffered on stdout to the operating system |

- **`flush` touches stdout only** — not stderr, not stdin.
- Writes through this module flush themselves, so a program that writes only
  through the four members above never needs it. It exists for completeness
  and for output the language's own buffering has touched.
- Success is `Ok`; an `Err` means the flush failed, with the same `message`
  contract as a write. The match pattern of the reference example:

```xulo
match std::io::flush() {
  Result::Ok(_) => null
  Result::Err(e) => {
    std::io::eprintln(e.message)
    null
  }
}
// runs: nothing on success; on a failed flush, the reason on stderr
```

## Terminal probes

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::io::stdinIsTTY` | `fn(): bool` | whether stdin is connected to a terminal |
| `std::io::stdoutIsTTY` | `fn(): bool` | whether stdout is connected to a terminal |

- Both probes are **total**: a `bool` for every call — never a `Result`,
  never `null`. A stream either is a terminal or is not, and the question is
  always answerable.
- `stdoutIsTTY` is the color guard: a program that writes escape sequences
  checks it first and writes plain text when the answer is `false`, so output
  piped to a file or another program stays clean.
- `stdinIsTTY` is the interactivity guard: when it is `false`, input comes
  from a pipe or a redirect, and a program prompts only when the answer is
  `true`.
- Stderr is deliberately not probed: no member of this module turns a write
  on or off by destination — the four writes are unconditional — and the
  diagnostic stream is written either way.

```xulo
if std::io::stdoutIsTTY() {
  std::io::println("colored")     // a terminal: escapes are welcome
} else {
  std::io::println("plain")       // piped: keep the bytes clean
}
```

## Line reading

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::io::readLine` | `fn(): Result<string?, Error>` | the next line of stdin without its terminator, or `Ok(null)` at end of input |

- The line is returned through its terminator: a trailing `\n`, or a
  trailing `\r\n` as the one terminator, is stripped. A final line that ends
  at end of input with no terminator is still a line, returned whole.
- End of input is **absence**: `Ok(null)` — the `null` inside `string?`, not
  an `Err` ([`README.md`](README.md) § 5.2). An empty line is `Ok("")` and
  never passes for end of input.
- Bytes that are not valid UTF-8 are an `Err`; the `message` names the byte
  offset at which validity ends (informative,
  [`../README.md`](../README.md) § 3.5).
- The call **blocks** until a line or end of input arrives — a blocking
  member in the sense of [`README.md`](README.md) § 5.6. There is no size
  limit: a line of any length is returned whole, or fails only as above.

```xulo
// stdin holds b"\n" — one empty line, then nothing
std::io::readLine()     // runs: Ok("")
std::io::readLine()     // runs: Ok(null) — end of input, distinct from the line above
// stdin holds b"tail" — a final line with no terminator
std::io::readLine()     // runs: Ok("tail")
```

Reading until end of input is a loop on absence:

```xulo
let mut done = false
while !done {
  match std::io::readLine() {
    Result::Ok(null) => done = true
    Result::Ok(line) => std::io::println(line)
    Result::Err(e) => {
      std::io::eprintln(e.message)
      done = true
    }
  }
}
```

## Stream reading

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::io::read` | `fn(): Result<string, Error>` | all of stdin, drained to end of input |

- The remaining stream is returned as one string; the call **blocks** until
  end of input (blocking member, [`README.md`](README.md) § 5.6). There is
  no size limit.
- End of input with no data is `Ok("")` — not `null`. `read` has no absence
  case: every success carries a string, empty or not.
- Bytes that are not valid UTF-8 anywhere in the stream are an `Err`,
  reported exactly as `readLine` reports them — the message names the byte
  offset (informative).

```xulo
// stdin is empty
std::io::read()         // runs: Ok("") — no absence; contrast readLine's Ok(null)
```

## Bounded reading

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::io::readN` | `fn(max: number): Result<string?, Error>` | up to `max` bytes of stdin, decoded as UTF-8; `Ok(null)` at end of input |

- `max` is a byte count in **`1 …= 16777216`** (16 MiB) — the same domain
  [`net.md`](net.md)'s `sock_read` names — and a fractional value counts as
  its truncation toward zero. A value outside that domain is `Err`, checked
  before the call blocks, so a bad `max` is reported whatever stdin holds.
  The domain is an `Err`, never a runtime failure.
- The call **blocks** until at least one byte arrives or input ends
  (blocking member, [`README.md`](README.md) § 5.6) and yields **up to**
  `max` bytes of what arrived: a short success is ordinary, and the next
  `readN` continues the stream where this one stopped. The read stops at a
  **character boundary** — an incomplete UTF-8 sequence at the cut stays in
  the stream for the next call, never split and never decoded.
- Decoding is strict, as in `readLine` and `read`: bytes that are not valid
  UTF-8 are an `Err`, the `message` naming the byte offset (informative,
  [`../README.md`](../README.md) § 3.5) — never a replacement character.
  [`net.md`](net.md)'s `sock_read` decodes lossily because a socket carries
  arbitrary bytes; stdin is text, as this module's other two readers
  already assume.
- End of input is **absence**: `Ok(null)`, the same shape as `readLine`.
  EOF sticks: every later `readN` keeps yielding `Ok(null)`.
- `read` and `readN` divide the same job: `read` takes all the rest in one
  string, `readN` takes it in pieces when the whole would be too much to
  hold. The loop on absence is the one under
  [`readLine`](#line-reading), with `chunk` in place of `line`.

```xulo
match std::io::readN(4096) {
  Result::Ok(null) => null             // end of input
  Result::Ok(chunk) => std::io::println(chunk)
  Result::Err(e) => std::io::eprintln(e.message)
}
std::io::readN(0)    // runs: Err — outside 1 …= 16777216, before any block
```

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | every member but the probes — `print` `println` `eprint` `eprintln` `flush` `readLine` `read` `readN` |
| Absence (`Ok(null)`) | `readLine` and `readN` at end of input |
| Total | `stdinIsTTY` `stdoutIsTTY` |
| Runtime failure | none — this module states no argument domain that stops the program (`readN`'s `max` domain is an `Err`, not a runtime failure) |

## Checked call examples

```xulo
std::io::print("x")                    // ok
use std::io as io;
io::println("y")                       // ok: the same member, aliased
std::io::print(42)                     // error[E0202]: expected `string`, found `int` — an int literal does not coerce to string outside interpolation
std::io::flush(1)                      // error[E0207]: expected 0 arguments, found 1
std::io::stdoutIsTTY()                 // ok: an ordinary bool — no Result
std::io::readN(4096)                   // ok: Result<string?, Error>
std::io::readN("4096")                 // error[E0202]: expected `number`, found `string`
use std::io as print;                  // error[E0601]: prelude intrinsic name — README.md § 2.3
std::io::writelines(["a", "b"])        // error[E0601]: no member `writelines`
```
