# `std::time`

`std::time` holds the clocks, the delay, and the calendar: ten functions, no
constants and no types — two wall-clock readings, two monotonic readings,
`sleep`, the one member of the whole library that returns a task, and five
members that render a timestamp as ISO 8601 text, read one back, and expose
its date fields. The readings and the calendar are total; `parse` is fallible
and `format` and `fromParts` state argument domains (Failure model). The
contract is host-independent ([`README.md`](README.md) § 5.5): the readings
come from the host, but the units, the epoch, the monotonicity guarantee, and
the UTC calendar are fixed here — the same timestamp renders, parses, and
decomposes identically on every machine.

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

## Calendar

The calendar members work in **UTC alone** — no host time zone, no DST, no
locale — and share one timestamp domain: the milliseconds `0 …= 253402300799999`,
from `1970-01-01T00:00:00.000Z` to `9999-12-31T23:59:59.999Z`. Leap seconds
are not represented: the count is ordinary Unix civil time, and `23:59:60Z`
does not parse.

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::time::format` | `fn(ms: u64): string` | the timestamp as ISO 8601 UTC — `2026-10-26T12:34:56.789Z` |
| `std::time::parse` | `fn(text: string): Result<u64, Error>` | the timestamp an ISO 8601 UTC string denotes |
| `std::time::parts` | `fn(ms: u64): { year: int, month: int, day: int, hour: int, minute: int, second: int, millisecond: int }` | the UTC date and time fields of a timestamp |
| `std::time::fromParts` | `fn(year: int, month: int, day: int, hour: int = 0, minute: int = 0, second: int = 0, millisecond: int = 0): u64` | the timestamp those UTC fields denote |
| `std::time::weekday` | `fn(ms: u64): int` | the day of the week — `0` Sunday … `6` Saturday |

- `format` emits one fixed layout: four-digit year, two-digit month, day,
  `T`, hour, minute, second, then **exactly three** fractional digits and
  `Z` — no templates and no locale. A `ms` above the domain — past
  `9999-12-31T23:59:59.999Z` — is a runtime failure
  (`spec/memory-and-runtime.md`), the argument-domain shape of
  `std::math::toFixed`'s `digits`; every other `u64` renders.
- `parse` accepts the layout `format` emits, plus two relaxations: `T` or a
  single space as the date/time separator, and a fraction of 1 to 9 digits
  (truncated to the millisecond — the output's precision). Field ranges are
  checked against the calendar (a nonexistent date like `2026-02-30`, a
  `23:59:60Z` leap second, or an out-of-range hour is rejected), and `Z` is
  required — an offset such as `+08:00` is not part of this version.
  Everything else — malformed text, a date before the Unix epoch, a date
  beyond `9999` — is `Err` of the **built-in** `Error`
  ([`README.md`](README.md) § 5.1), never a panic and never an absence.
  For every `ms` in the domain, `parse(format(ms))` is `Ok(ms)`.
- `parts` is total for every `u64`, the calendar's domain included: a
  timestamp `format` refuses still decomposes — the `year` field is an
  `int`, large enough for any value the clock can return. The fields are
  UTC: `month` runs `1 …= 12`, and `millisecond` runs `0 …= 999`.
- `fromParts` inverts `parts`: the fields must name a real instant inside
  the domain — a month of `1 …= 12`, a day within that month (leap years
  included), `hour` `0 …= 23`, `minute` and `second` `0 …= 59`,
  `millisecond` `0 …= 999`, and a date at or after the Unix epoch. Any
  other field set is a **runtime failure**; the time fields default to `0`,
  so `fromParts(2026, 10, 26)` is midnight UTC.
- `weekday` is total for every `u64`, domain or not: the epoch itself was a
  Thursday, so `weekday(0) == 4`.

```xulo
let noon = std::time::fromParts(2026, 10, 26, 12)
print(std::time::format(noon))              // runs: 2026-10-26T12:00:00.000Z
match std::time::parse("2026-10-26T12:00:00.000Z") {
    Result::Ok(ms) => print(ms == noon)     // runs: true
    Result::Err(e) => print(e.message)
}
let f = std::time::parts(noon)
print(`${f.year}-${f.month}-${f.day}`)      // runs: 2026-10-26
print(std::time::weekday(noon))             // runs: 1 — a Monday
std::time::fromParts(2026, 2, 30)           // runtime failure: the calendar has no such date
std::time::format(253402300800000)          // runtime failure: past 9999-12-31T23:59:59.999Z
```

### Cost (informative)

`parts` runs one pass of fixed integer arithmetic — milliseconds divided by
86400000 for a day count, then year, month, and time of day resolved by a
handful of further divisions — with no host call and no work proportional
to the fields returned. A single-field reader would run that same pass on
every call: a pure function has no object to cache the result on, and the
cache that makes JavaScript's `Date` getters cheap is the boxed `Date`
itself, which this module deliberately does not have. `weekday` is the same
day count reduced by 7. An implementation may inline these members and
eliminate repeated calls in a hot loop; what this specification guarantees
is semantics (§ 5.5, § 5.7), not timings.

## Failure model

| Kind | Members |
|------|---------|
| Total | `now`, `nowNanos`, `monotonic`, `monotonicNanos`, `sleep`, `parts`, `weekday` |
| Fallible (`Result`) | `parse` — `Result<u64, Error>`, the built-in error |
| Runtime failure | `format` with `ms` past `9999-12-31T23:59:59.999Z`; `fromParts` with fields outside the calendar or before the Unix epoch |

The readings, `sleep`, `parts`, and `weekday` cannot fail: there is no
absence in this module, and `sleep`'s clamping of an oversized delay is a
settling guarantee, not a failure. `parse` is the module's only fallible
member, and its `Err` carries the **built-in** `Error` — the default rule,
with no module error type ([`README.md`](README.md) § 5.1). `format` and
`fromParts` state argument domains that stop the program when exceeded, the
same shape as `std::math::toFixed`'s `digits`. The only check-time error
that involves these members is discarding `sleep`'s task in statement
position (`E0218`), reported by the checker before the program runs, not a
runtime failure of the member.

## Checked call examples

```xulo
std::time::now()                      // ok
std::time::now() > 0                  // ok: true on any host after 1970
std::time::monotonicNanos()           // ok
std::time::now(1)                     // error[E0207]: expected 0 arguments, found 1
std::time::fromParts(2026, 10, 26)    // ok: hour … millisecond default to 0
std::time::fromParts(2026, 10)        // error[E0207]: expected 3 to 7 arguments, found 2
std::time::parse(2026)                // error[E0202]: expected `string`, found `int`
std::time::sleep("100")               // error[E0202]: expected `u64`, found `string`
async fn ok() {
  await std::time::sleep(1000)        // ok: awaited inside an async body
}
let nap = std::time::sleep(1000)      // ok: the task is bound, not discarded
std::time::sleep(1000)                // error[E0218]: expected `unit`, found `Task<unit>`
Time::now()                           // error[E0607]: `Time` lives under `std`: use `std::time::now`
```
