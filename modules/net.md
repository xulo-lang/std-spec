# `std::net`

`std::net` holds the TCP socket surface: ten functions that bind, connect,
accept, read, write, shut down, close, and name sockets — every socket
addressed by an opaque numeric **handle** that the openers issue and every
member validates ([`../README.md`](../README.md) § 5). Version 0.1 is TCP
over IPv4/IPv6 only: no UDP, no TLS, no byte-level read API — those are
roadmap candidates ([`../README.md`](../README.md) § 1.2). HTTP is not
re-implemented here; it is a module above these primitives
([`http.md`](http.md)). The module is
host-dependent ([`README.md`](README.md) § 5.5): which addresses the host
resolves, whether a bind is permitted, how large a listen backlog the
kernel grants, and how a refused connection reports itself are the host's
affair — the contract below is the shape of every answer. Failures are
ordinary values ([`README.md`](README.md) § 5.1): a dead handle, a
wrong-kind handle, a refused connection, or an expired budget arrives as
`Result::Err`, and end of stream arrives as the `null` inside `string?`.

The qualified path needs no import:

```xulo
std::net::tcpListen("127.0.0.1:8080")   // ok: Result<number, Error>
std::net::sockClose(1)                  // ok: Result<unit, Error>
```

Import forms are the six of [`README.md`](README.md) § 2.2; the face is the
ten functions — no constants and no types ([`README.md`](README.md) § 3):

```xulo
use std::net::*;                         // the ten functions
use std::net as net;
net::tcpConnect("127.0.0.1:8080")       // ok: same member, aliased
use std::net::{sockRead, sockWrite};
```

## Handles

- The three openers — `tcpListen`, `tcpConnect`, `tcpAccept` — yield the
  handle of a host socket: an opaque **number**, meaningless outside this
  module and accepted only by `std::net` members. Handles are per-process
  names, allocated from `1` upward and **never reused**: after `sockClose`
  the old number stays dead, and a fresh socket always gets a fresh number,
  so a stale handle never names another socket's traffic.
- Version 0.1 has two kinds of handle: a **listener** (`tcpListen`) and a
  **stream** (`tcpConnect`, and every handle `tcpAccept` yields). A
  member given the wrong kind is `Err`, saying so — never a panic, never
  the operation half applied: `tcpAccept` on a stream is `Err`;
  `sockRead`, `sockWrite`, `sockShutdown`, `sockShutdownWrite`, and
  `sockPeerAddr` on a listener are `Err` (a listener has no peer and no
  data); `sockClose` and
  `sockLocalAddr` accept **either kind**.
- **Every member validates its handle first.** A number this process was
  never issued, an already-closed handle, and a wrong-kind handle are each
  a clean `Err` — the whole failure inventory for handles — whatever else
  the call was asked to do.

## Addresses

- Every address is one string in `"host:port"` form: an IPv4 literal
  `127.0.0.1:8080`, a **DNS name** the host resolves (`localhost:8080`
  works), or an IPv6 literal in brackets `[::1]:8080`; the port is decimal,
  `0 …= 65535`. A string that does not resolve or is not a `"host:port"` at
  all is `Err`, never a panic. An address resolving to several socket
  addresses is tried in turn; the last candidate's failure is the call's
  `Err`.
- Port `0` asks the host for an **ephemeral** port: `tcpListen` binds one,
  and `sockLocalAddr` reads the assigned `"ip:port"` back — a string that
  can be handed straight to `tcpConnect`.

## Timeouts

- The four calls that can wait for the outside world — `tcpConnect`,
  `tcpAccept`, `sockRead`, `sockWrite` — take an optional trailing
  `timeout: u64? = null`: a budget in **milliseconds**, measured from the
  call's start. `null` waits without limit, the behavior before `timeout`
  existed, kept as the default.
- An expired budget is `Result::Err`, a fourth class beside the handle, the
  address, and the host classes. It is not a runtime failure, and it never
  arrives as absence: `Ok(null)` keeps meaning end of stream and nothing
  else.
- The handle a timed-out call was given stays open and usable —
  `tcpAccept` may be tried again on the same listener, `sockRead` again
  on the same stream.
- A program that polls with a budget cannot tell an expired budget from
  the other `Err` classes: version 0.1 has no structured error kind, every
  `Err` has the same shape, and message text is not a discriminator
  ([`README.md`](README.md) § 5.1). The distinction costs nothing at the
  call site when a budget only exists to stop a hang.
- What a timed-out call did is defined by the call: nothing is consumed by
  `sockRead`, no handle is issued by `tcpConnect`, and only `sockWrite`
  can leave a prefix handed to the host (see [Writing](#writing)).
- A call that finishes inside its budget never reports a timeout, however
  close the finish.
- `tcpListen` takes no budget: binding a local address does not wait on
  the outside world.
- The budget is a property of the call, not of the socket. Version 0.1 has
  no per-handle default budget; that is a roadmap candidate, and an
  explicit `timeout` would always override such a default. Until then an
  omitted `timeout` waits without limit.

## The openers

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::tcpListen` | `fn(addr: string): Result<number, Error>` | bind and listen on `addr`; yield the listener's handle |
| `std::net::tcpConnect` | `fn(addr: string, timeout: u64? = null): Result<number, Error>` | connect to `addr`, bounded by `timeout`; yield the stream's handle |
| `std::net::tcpAccept` | `fn(handle: number, timeout: u64? = null): Result<number, Error>` | wait for the next incoming connection on a listener, bounded by `timeout`; yield its **new** stream handle |

- `tcpListen` uses the host's default listen backlog; on success the
  handle names a listener until `sockClose` releases it, and it stays open
  across any number of `tcpAccept` calls. `Err` classes: the address does
  not resolve or is malformed; every resolved address fails to bind —
  address in use, a privileged port, or another host refusal.
- `tcpConnect` **blocks** until the connection completes or fails (a
  blocking member, [`README.md`](README.md) § 5.6). `Err` classes: the
  address does not resolve or is malformed; the connection is refused,
  unreachable, or fails at the host for another reason; the budget
  expires. The budget covers name resolution and every candidate address;
  an expired budget issues no handle.
- `tcpAccept`'s operand must be a live listener. It **blocks** until a
  client arrives (§ 5.6) and yields a handle distinct from every other;
  the listener itself stays open and can accept again. `Err` classes: the
  handle is not open; the handle is a stream, not a listener; the host's
  accept fails; the budget expires. A timed-out accept leaves the listener
  open, so a server loop may simply accept again.

## Reading

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::sockRead` | `fn(handle: number, max: number, timeout: u64? = null): Result<string?, Error>` | block for up to `max` bytes of input, decoded as UTF-8, bounded by `timeout` |

- `max` is a byte count in **`1 …= 16777216`** (16 MiB); a fractional value
  counts as its truncation toward zero. A value outside that domain — zero,
  negative, non-finite, or above the cap — is `Err`, checked **before** the
  handle is looked up, so a bad `max` is reported whatever the handle
  names. The domain is an `Err`, never a runtime failure.
- The call **blocks** until at least one byte arrives (§ 5.6) and yields
  **up to** `max` bytes of what has arrived: a short success is ordinary,
  and the next `sockRead` continues the stream where this one stopped.
  The budget bounds the wait for the **first** byte; input already
  available is returned without consulting it. A timed-out read consumes
  nothing, so the next call starts where this one did. `Err` classes: the
  handle is not open; the handle is a listener; the host reports an I/O
  error; the budget expires.
- Bytes are decoded **lossily** as UTF-8: a sequence that is not valid
  UTF-8 becomes U+FFFD replacement characters. Each call decodes its own
  bytes independently, so a multi-byte character **split across two reads**
  is destroyed in both — the leading fragment decodes to U+FFFD in the
  first call, the orphaned continuation bytes to U+FFFD at the start of the
  second. The consequence: the stream is a **byte stream dressed as a
  string**, not a channel safe for arbitrary binary; frame messages when
  text matters.
- **End of stream** — the peer closed, or its write half was shut down — is
  absence: `Ok(null)`, the `null` inside `string?`, never an `Err`. EOF
  sticks: every later `sockRead` of that stream keeps yielding `Ok(null)`.
  A host I/O error on the stream is `Err`. Until the peer closes, reading
  is a loop on absence exactly like [`io.md`](io.md)'s line loop: match
  `Ok(null)` as the end, `Ok(chunk)` as data.

## Writing

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::sockWrite` | `fn(handle: number, data: string, timeout: u64? = null): Result<number, Error>` | write `data`'s UTF-8 bytes, bounded by `timeout`; yield the byte count |

- `data` is encoded as UTF-8 and the call **blocks** until every byte is
  handed to the host (§ 5.6): it **loops to completion** and never yields
  a short count the caller must finish. The `Ok` payload is always the
  full UTF-8 byte length of `data` — bytes, not characters, so `"héllo"`
  counts 6. An empty `data` is success with `0`.
- `Err` classes: the handle is not open; the handle is a listener; the
  stream is no longer writable (the peer is gone, or the handle was shut
  down); the host reports an I/O error; the budget expires. A timed-out
  write bounds the whole loop and may follow a prefix already handed to the
  host: the handle stays open, but the caller cannot assume the write was
  complete, so frame above this layer when that matters.

## Shutting down and closing

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::sockShutdown` | `fn(handle: number): Result<unit, Error>` | half-close a stream in **both** directions |
| `std::net::sockShutdownWrite` | `fn(handle: number): Result<unit, Error>` | half-close a stream's **writing** direction, leaving reading open |
| `std::net::sockClose` | `fn(handle: number): Result<unit, Error>` | release the handle and close the socket |

- `sockShutdown` shuts a stream down for reading **and** writing: the peer
  sees end of stream, and this side's reads see it too — a later
  `sockRead` of the handle yields `Ok(null)`. The handle **stays open** in
  the module's registry: a later `sockClose` still succeeds and is what
  actually releases the socket, while a later `sockWrite` is `Err` (the
  stream is no longer writable). It is exactly `sockShutdownWrite` plus a
  local read-side shutdown. `Err` classes: the handle is not open; the
  handle is a listener.
- `sockShutdownWrite` closes only the **writing** half: the peer sees end
  of stream, while this side's `sockRead` keeps working until the peer
  closes, and then yields `Ok(null)`. A later `sockWrite` is `Err`; the
  handle stays open, and `sockClose` still releases it. This is the shape
  of a request-then-read exchange — write the request, half-close, read the
  answer until EOF — and it is the composition point for protocols that
  signal "request complete" with a FIN. `Err` classes: the handle is not
  open; the handle is a listener.
- `sockClose` releases the handle and closes the socket it named — a
  listener or a stream. From that moment every member on the number is
  `Err`, `sockClose` itself included: **a second close is `Err`**, not an
  idempotent success. Handles are never reused, so the dead number never
  becomes another socket's name.

## Addresses of a handle

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::sockLocalAddr` | `fn(handle: number): Result<string, Error>` | the handle's local `"ip:port"` |
| `std::net::sockPeerAddr` | `fn(handle: number): Result<string, Error>` | a stream's remote `"ip:port"` |

- `sockLocalAddr` answers for **either kind** of handle: the address the
  listener is bound to, or the stream's local end — including the ephemeral
  port assigned through port `0`, which is how a program learns where to
  connect. `sockPeerAddr` answers only for a **stream**: a listener is
  `Err`, and so is a handle that is not open.
- Both yield the host's `"ip:port"` rendering — bracketed for IPv6 — and
  that string is itself a well-formed argument for `tcpListen` or
  `tcpConnect`.

## A loopback round trip

Listen on an ephemeral loopback port, read the port back, connect, accept,
then echo in both directions — every call matched, because every call is
fallible:

```xulo
match std::net::tcpListen("127.0.0.1:0") {
  Result::Ok(server) => match std::net::sockLocalAddr(server) {
    Result::Ok(addr) => match std::net::tcpConnect(addr) {
      Result::Ok(client) => match std::net::tcpAccept(server) {
        Result::Ok(accepted) => {
          match std::net::sockWrite(client, "ping") {
            Result::Ok(n) => print(n)                    // runs: 4 — bytes
            Result::Err(e) => print(e.message)
          }
          match std::net::sockRead(accepted, 64) {
            Result::Ok(null) => print("eof")
            Result::Ok(text) => print(text)              // runs: ping
            Result::Err(e) => print(e.message)
          }
          match std::net::sockWrite(accepted, "pong") {
            Result::Ok(n) => print(n)                    // runs: 4 — bytes
            Result::Err(e) => print(e.message)
          }
          match std::net::sockRead(client, 64) {
            Result::Ok(null) => print("eof")
            Result::Ok(text) => print(text)              // runs: pong
            Result::Err(e) => print(e.message)
          }
          std::net::sockClose(client)                   // ok
          std::net::sockClose(accepted)                 // ok
          std::net::sockClose(server)                   // ok
        }
        Result::Err(e) => print(e.message)
      }
      Result::Err(e) => print(e.message)
    }
    Result::Err(e) => print(e.message)
  }
  Result::Err(e) => print(e.message)
}
```

The handshake is in program order — listen, read the address, connect,
accept — because `tcpConnect` completes through the listen backlog
*before* any `tcpAccept` blocks (see below).

## Small focused cases

On the three handles of a completed handshake — the listener `server`, the
connected `client`, and its accepted peer `accepted`:

```xulo
match std::net::sockWrite(client, "héllo") {
  Result::Ok(n) => print(n)              // runs: 6 — the UTF-8 byte count
  Result::Err(e) => print(e.message)
}

std::net::sockRead(accepted, 64)        // runs: Ok("héllo") — buffered data arrives before EOF
std::net::sockShutdown(client)          // ok: both directions half-closed
std::net::sockRead(accepted, 64)        // runs: Ok(null) — the peer's close is absence, not a failure
std::net::sockRead(accepted, 64)        // runs: Ok(null) — EOF sticks
std::net::sockRead(accepted, 0)         // runs: Err — max outside 1 …= 16777216
std::net::sockRead(999999, 8)           // runs: Err — the handle is not open
std::net::sockWrite(server, "x")        // runs: Err — a listener, not a stream

std::net::sockClose(accepted)           // ok: the handle is released
std::net::sockClose(accepted)           // runs: Err — already closed
```

## HTTP over these primitives

HTTP/1.1 is CRLF-delimited text carried by a TCP stream; nothing in this
module is HTTP-aware, and everything an HTTP exchange needs is here:

| HTTP step | Members |
|-----------|---------|
| fetch a URL | `tcpConnect`, then `sockWrite` the request head and body |
| read an answer to its end | `sockRead` in a loop, `Ok(null)` at end of stream |
| signal "request complete" | `sockShutdownWrite`, then read until `Ok(null)` |
| serve a URL | `tcpListen`, `tcpAccept`, then `sockRead` and `sockWrite` |
| bound any of those waits | the `timeout` argument ([Timeouts](#timeouts)) |

[`std::http`](http.md) packages exactly this — request and response types,
a client, and a sequential serving loop — and owns the protocol rules
(framing, size caps, statuses) that this module deliberately does not. A
TLS layer would be a further module above both; `https` is not reachable
from these primitives.

## Blocking and asynchrony

- All ten members are **synchronous** ([`README.md`](README.md) § 5.6): the
  call returns when the host operation completes. The members that can wait
  on the outside world — the openers `tcpListen`, `tcpConnect`,
  `tcpAccept` and the transfers `sockRead` and `sockWrite` — are the
  **blocking members** of the terminology table; while one waits, no other
  task makes progress: the cooperative scheduler is frozen for the
  duration. A `timeout` bounds the freeze ([Timeouts](#timeouts)); without
  one the wait may be unbounded.
- The freeze reaches single-threaded programs too, which makes the
  handshake order load-bearing: `tcpConnect` completes through the listen
  backlog *before* `tcpAccept` is called, so a program that calls
  `tcpAccept` first — with no connection on its way — waits until a
  `timeout` expires, or forever when it passes none.
- Version 0.1 has no asynchronous variant of any member; that is a roadmap
  candidate ([`../README.md`](../README.md) § 1.2), not part of this
  version.

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | every member — `tcpListen` `tcpConnect` `tcpAccept` `sockRead` `sockWrite` `sockShutdown` `sockShutdownWrite` `sockClose` `sockLocalAddr` `sockPeerAddr` |
| Absence (`Ok(null)`) | `sockRead` at end of stream |
| Total | none |
| Runtime failure | none — `sockRead`'s `max` domain is an `Err`, not a runtime failure; an expired `timeout` is an `Err` too |

Failures are ordinary `Result::Err` values carrying `Error { message }`;
message text is informative ([`README.md`](README.md) § 5.1,
[`../README.md`](../README.md) § 3.5), and the contract is the `Err` shape
plus the failure classes each section above enumerates.

## Checked call examples

```xulo
std::net::tcpListen("127.0.0.1:0")      // ok: Result<number, Error>
std::net::tcpConnect("127.0.0.1:8080", 500)  // ok: Result<number, Error>
std::net::tcpAccept(1, 250)             // ok: Result<number, Error>
std::net::sockRead(1, 16, 25)           // ok: Result<string?, Error>
std::net::sockRead(1)                   // error[E0207]: expected 2 to 3 arguments, found 1
std::net::sockRead("h", 16)             // error[E0202]: expected `number`, found `string`
std::net::sockShutdownWrite(1)         // ok: Result<unit, Error>
std::net::open("x")                      // error[E0601]: no member `open`
Net::tcpListen("127.0.0.1:0")           // error[E0607]: `Net` lives under `std`: use `std::net::tcpListen`
```
