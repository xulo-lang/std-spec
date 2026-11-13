# `std::hint`

`std::hint` holds the members that tell an optimizing implementation how code
is expected to behave without changing what the program observes: `black_box`
passes a value through as an opaque barrier, `likely` and `unlikely` mark the
expected outcome of a branch, and `spin_loop` marks a busy-wait loop. The
values the program observes are exactly the operands and conditions supplied —
the hints affect generated code or the processor, never meaning. The face is
exactly those four functions; the module holds no constants and no types.

The module is total, synchronous, stateless, and host-independent: it reads
nothing, writes nothing, and touches no clock, file, socket, environment, or
process. Every call is defined for every admissible argument, and no member
returns `Result` or reports absence (§ 5.1 and § 5.2 of
[`README.md`](README.md)). The same operand or condition gives the same value
on every machine; a hint changes timing and code shape, not the result
(§ 5.5 and § 5.7 of [`README.md`](README.md)).

The qualified path needs no import:

```xulo
std::hint::black_box(7)                // ok: 7
std::hint::likely(n > 0)               // ok: the condition's value, unchanged
std::hint::spin_loop()                 // ok: unit
```

Import forms are the six of [`README.md`](README.md) § 2.2:

```xulo
use std::hint::*;                      // binds black_box likely spin_loop unlikely
use std::hint::{black_box, likely};
use std::hint as h;
h::unlikely(err)
```

## The members

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::hint::black_box` | `fn(value: T): T` | the operand, unchanged; `T` is the operand's static type |
| `std::hint::likely` | `fn(condition: bool): bool` | the condition, unchanged; hints that it is usually true |
| `std::hint::spin_loop` | `fn(): unit` | a processor hint for a busy-wait loop; no observable effect |
| `std::hint::unlikely` | `fn(condition: bool): bool` | the condition, unchanged; hints that it is usually false |

`black_box`, `likely`, and `unlikely` take exactly one argument; `spin_loop`
takes none. Any other count is `E0207` ([`README.md`](README.md) § 4).

### `black_box`

The `T` in the signature is a metavariable for the operand's static type: the
call's type is exactly the type of `value` — not a fixed type, and not `any`.
Every type the language can produce is admissible; `black_box` neither
constrains nor converts the value. `T` is derived from the argument at each
call and is never written by the caller: the member declares no type parameter,
and an explicit type argument is rejected — `std::hint::black_box<int>(3)` is
`E0807` ([`README.md`](README.md) § 4). The derived type is the operand's
exact type, and the ordinary assignability rules apply to the result:

```xulo
let n = std::hint::black_box(3)            // ok: int
let m = n + 1                              // ok: int arithmetic
let s = std::hint::black_box("hi")         // ok: string
print(s + "!")                             // ok: string concatenation
let xs = std::hint::black_box([1, 2])      // ok: List<int>
print(xs.len)                              // ok: 2
let bad: string = std::hint::black_box(3)  // error[E0201]: expected `string`, found `int`
```

`black_box` is the identity: it returns the operand itself, with the operand's
type. It does not inspect, convert, normalize, or modify the value, and its
result denotes the same value as the operand. The argument is evaluated
exactly once, in the ordinary argument position
([`xulo-spec/expressions/calls.md`](../xulo-spec/expressions/calls.md)).

Alongside the identity, the member is an **optimization barrier**. An
optimizing implementation MUST treat the result as a value it cannot predict
from the operand's text: it MUST NOT replace the call with the operand's
compile-time value, nor use that known value to fold, precompute, or simplify
the computation around the call. This is a conformance requirement on
generated code, not an observable semantic guarantee: the barrier changes
neither the value nor any other behavior a program can test, only the time and
resources the computation takes. An implementation with no optimizer, such as
the interpreter, returns the operand directly and satisfies the requirement
trivially.

The intended use is a benchmark or a measurement, where a pure computation
must not be removed or precomputed, as in this loop:

```xulo
fn sum_loop(n: int): int {
  let mut sum = 0
  let mut i = 0
  while i < n {
    sum = std::hint::black_box(sum + std::hint::black_box(i))
    i = i + 1
  }
  sum
}
```

### `likely` and `unlikely`

`likely` and `unlikely` are identities on their operand: each evaluates its
`condition` exactly once and returns that value, so `likely(c)` and `unlikely(c)`
have the same value as `c` and are usable wherever the `bool` is usable. They
differ only in the hint they carry:

- `likely(condition)` tells the implementation that `condition` is usually
  true, so the path taken when it is true should be the common, fast path.
- `unlikely(condition)` tells the implementation that `condition` is usually
  false, so the path taken when it is true should be the exceptional path.

The usual spelling places the call in the condition of an `if` or `while`:

```xulo
if std::hint::unlikely(err != null) {
  handle_error(err)
}

while std::hint::likely(i < n) {
  i = i + 1
}
```

The hint is advice: a conforming implementation MAY use it to arrange branches
and cold code, and MAY ignore it. It never changes the condition, the branch
taken, or any result. A non-`bool` argument is `E0202`.

### `spin_loop`

`spin_loop()` takes no argument and returns `unit`. It signals to the processor
that the calling thread is busy-waiting — typically by emitting the target's
spin-wait instruction (`PAUSE` on x86, `YIELD` on some others) — so the core
can reduce contention and power while the loop retries. On a target with no
such instruction it is a no-op.

```xulo
while !ready() {
  std::hint::spin_loop()
}
```

`spin_loop` is a processor hint, not a synchronization primitive and not a
scheduler call: it neither acquires memory ordering nor guarantees that the
loop makes progress, and it does not yield the thread. A program that needs to
wait for another task uses the concurrency facilities of the language, or a
blocking member ([`README.md`](README.md) § 5.6) such as
`std::io::read_line`; `spin_loop` only makes a deliberate busy-wait cheaper.

## Failure model

| Kind | Members |
|------|---------|
| Total | `black_box`, `likely`, `spin_loop`, `unlikely` |
| Runtime failure | none |
| Fallible (`Result`) | none |

No member meets the outside world, so there is no failure for § 5.1's `Result`
to carry; every admissible argument is defined, so there is no argument domain
for § 5.3's runtime failure either. The only errors are static — a wrong
argument count (`E0207`), a non-`bool` condition for `likely` or `unlikely`
(`E0202`), an unknown member (`E0601`), explicit type arguments (`E0807`), or
a bare or capital head (`E0607`) — each with the code
[`README.md`](README.md) § 4 assigns.

## Checked call examples

```xulo
std::hint::black_box(7)                // ok: 7
std::hint::black_box(2.5)              // ok: 2.5
std::hint::black_box("x")              // ok: "x"
std::hint::black_box(true)             // ok: true
std::hint::black_box(null)             // ok: null
std::hint::black_box([1, 2, 3])        // ok: the same list
std::hint::likely(ready)               // ok: the value of `ready`
std::hint::unlikely(failed)            // ok: the value of `failed`
std::hint::spin_loop()                 // ok: unit
std::hint::black_box()                 // error[E0207]: expected 1 argument, found 0
std::hint::black_box(1, 2)             // error[E0207]: expected 1 argument, found 2
std::hint::likely()                    // error[E0207]: expected 1 argument, found 0
std::hint::likely(1)                   // error[E0202]: expected `bool`, found `int`
std::hint::spin_loop(1)                // error[E0207]: expected 0 arguments, found 1
std::hint::black_box<int>(3)           // error[E0807]: `std::hint::black_box` accepts no type arguments
std::hint::empty(1)                    // error[E0601]: no member `empty`
Hint::black_box(1)                     // error[E0607]: `Hint` lives under `std`: use `std::hint::black_box`
```
