# `std::io`

`std::io` holds the process's standard streams — stdout, stderr, and stdin —
as the real, running operations a program talks to: every write flushes
before it returns, and the two reads drain stdin. Every member is fallible:
each returns a `Result`, and a closed or broken stream arrives in the `Err`
slot as data, never as a panic ([`README.md`](README.md) § 5.1). The module
is host-dependent ([`README.md`](README.md) § 5.5): which bytes the streams
carry is the process environment's affair.

The qualified path needs no import:

```xulo
std::io::print("io: ")                 // ok: stdout, no terminator, flushed
std::io::println("hello stdout")       // ok: a newline appended, flushed
std::io::eprintln("io: hello stderr")  // ok: the same write, on stderr
```

Import forms are the six of [`README.md`](README.md) § 2.2, with this
module's one deviation: the face is five of the seven exported names —
`eprint` `eprintln` `flush` `readLine` `read` — and `print`/`println` are
withheld from `use` because their local names are prelude intrinsics
(`E0601`). The rule and its reason live in [`README.md`](README.md) § 3
(with § 2.3's rejection table); what matters here is that the two members
behind it are the same flushing writes:

```xulo
use std::io as io;
io::print("prompt: ")            // ok: exactly std::io::print — flushed before it returns
use std::io::*;                  // binds the face: eprint eprintln flush readLine read
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

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | every member — `print` `println` `eprint` `eprintln` `flush` `readLine` `read` |
| Absence (`Ok(null)`) | `readLine` at end of input |
| Total | none |
| Runtime failure | none — this module states no argument domain that stops the program |

## Checked call examples

```xulo
std::io::print("x")                    // ok
use std::io as io;
io::println("y")                       // ok: the same member, aliased
std::io::print(42)                     // error[E0202]: expected `string`, found `int` — an int literal does not coerce to string outside interpolation
std::io::flush(1)                      // error[E0207]: expected 0 arguments, found 1
use std::io as print;                  // error[E0601]: prelude intrinsic name — README.md § 2.3
std::io::writelines(["a", "b"])        // error[E0601]: no member `writelines`
```
