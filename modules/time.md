# `std::time`

`std::time` holds the clocks and the delay: five functions, no constants and
no types — two wall-clock readings, two monotonic readings, and `sleep`, the
one member of the whole library that returns a task. Every member is total:
there is no `Result`, no absence, and no runtime-failure domain in this
module. The contract is host-independent ([`README.md`](README.md) § 5.5):
the readings come from the host, but the units, the epoch, and the
monotonicity guarantee are fixed here.

Units are milliseconds for `now`, `monotonic`, and `sleep`, and nanoseconds
for the members whose name ends in `Nanos`; the unit is part of each member's
meaning, never a type name ([`README.md`](README.md) § 5.4).

This module is the former prelude `Time` namespace moved under `std`; the
migration is tracked in [`../README.md`](../README.md) § 1.3, and the
prelude spelling `Time::now()` is rejected (`E0607`, see
[`README.md`](README.md) § 2.5). The qualified path needs no import:

```xulo
std::time::now()                       // ok: milliseconds since the Unix epoch
std::time::now() > 0                   // ok: true on any host after 1970
```

Import forms are the six of [`README.md`](README.md) § 2.2:

```xulo
use std::time::*;               // every function of the face
use std::time as t;
t::monotonic()                  // ok: an ordinary u64 reading
use std::time::{now, sleep};
```

## Wall clock

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::time::now` | `fn(): u64` | milliseconds since the Unix epoch (wall clock) |
| `std::time::nowNanos` | `fn(): u64` | nanoseconds since the Unix epoch (wall clock) |

- `now` and `nowNanos` read the **wall clock**: the count of milliseconds or
  nanoseconds that has elapsed since 1970-01-01T00:00:00Z. They are the right
  choice for timestamps — the time of day a record was written, the stamp in
  a log line — and the wrong choice for measuring how long something took: a
  wall clock can be stepped by the host (NTP correction, manual change), so
  two readings may not order with real elapsed time.
- Both results are unsigned. A host whose clock reads a time *before* the
  Unix epoch yields `0` — never a negative number and never a failure — so a
  reading from such a host is indistinguishable from the epoch itself.

```xulo
let stampMs = std::time::now()          // ok: a wall-clock timestamp
let stampNs = std::time::nowNanos()     // ok: the same reading in nanoseconds
```

## Monotonic clock

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::time::monotonic` | `fn(): u64` | milliseconds on the monotonic clock |
| `std::time::monotonicNanos` | `fn(): u64` | nanoseconds on the monotonic clock |

- `monotonic` and `monotonicNanos` read a clock that **never runs backwards**
  and is unaffected by wall-clock changes: a reading taken after another one
  is greater than or equal to it, always.
- The origin of the monotonic clock is **arbitrary** — it is fixed once per
  process at the first reading — so an absolute value carries no meaning. Only
  the *difference* between two readings is meaningful, which makes these
  members the right choice for elapsed time, debounce windows, and benchmarks.
- The elapsed time between two readings is their difference; the unit of the
  difference is the unit of the members it was taken with.

```xulo
let start = std::time::monotonicNanos()     // ok
// ... work ...
let end = std::time::monotonicNanos()       // ok
let elapsed = end - start                   // ok: nanoseconds elapsed, never negative
```

### Nanosecond fidelity (informative)

A nanosecond reading of the current epoch is about 1.7 × 10¹⁸, well above
2⁵³ — the largest integer binary64 represents exactly. The reference
interpreter stores every number as binary64, so it rounds each nanosecond
reading to the nearest representable value, a step of about 256 ns at that
magnitude, while the compiled path carries the reading as an exact 64-bit
integer. This is a fidelity limitation of the reference interpreter, not of
this contract: a conforming implementation preserves the full integer value
of every clock reading.

## Sleep

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::time::sleep` | `fn(ms: u64): Task<unit>` | a task that settles once at least `ms` milliseconds have elapsed |

- `sleep` produces a `Task<unit>` that settles once **at least** `ms`
  milliseconds have elapsed — never earlier, possibly later when the host is
  busy. Awaiting it yields `unit`.
- The call itself returns immediately with the task; it does not block the
  caller. It is the one member of the library that returns a task, and it
  MUST be awaited inside an `async` body: `await std::time::sleep(1000)`.
  `await` is legal only inside an `async` body
  (`spec/expressions/async-expressions.md`).
- **A delay beyond what the host can represent is clamped to the host's
  maximum** — the task settles at that maximum instead of failing. Clamping
  is not a failure domain: every input yields a task.
- Discarding the task in statement position — writing `std::time::sleep(1000)`
  as a statement — is a check-time error (`E0218`), because the value is not
  `unit` and the discard is not one of the rule's exceptions
  (`spec/statements/expression-statements.md`). Bind it, `await` it, or pass
  it on.

```xulo
async fn later() {
  await std::time::sleep(1000)        // ok: suspends the body for at least 1 s
  std::io::println("1 second later")
}
```

## Failure model

| Kind | Members |
|------|---------|
| Total | every member — `now`, `nowNanos`, `monotonic`, `monotonicNanos`, `sleep` |
| Runtime failure | none |
| Fallible (`Result`) | none |

No `std::time` member can fail at runtime: there is no `Err`, no absence, and
no argument domain that stops the program — `sleep`'s clamping of an
oversized delay is a settling guarantee, not a failure. The only error that
involves these members is a **check-time** one: discarding `sleep`'s task in
statement position is `E0218`, reported by the checker before the program
runs, not a runtime failure of the member.

## Checked call examples

```xulo
std::time::now()                      // ok
std::time::now() > 0                  // ok: true on any host after 1970
std::time::monotonicNanos()           // ok
std::time::now(1)                     // error[E0207]: expected 0 arguments, found 1
std::time::sleep("100")               // error[E0202]: expected `u64`, found `string`
async fn ok() {
  await std::time::sleep(1000)        // ok: awaited inside an async body
}
let nap = std::time::sleep(1000)      // ok: the task is bound, not discarded
std::time::sleep(1000)                // error[E0218]: expected `unit`, found `Task<unit>`
Time::now()                           // error[E0607]: `Time` lives under `std`: use `std::time::now`
```
