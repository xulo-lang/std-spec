# `std::net`

`std::net` holds the TCP socket surface: nine functions that bind, connect,
accept, read, write, shut down, close, and name sockets — every socket
addressed by an opaque numeric **handle** that the openers issue and every
member validates ([`../README.md`](../README.md) § 5). Version 0.1 is TCP
over IPv4/IPv6 only: no UDP, no TLS, no HTTP surface — those are roadmap
candidates ([`../README.md`](../README.md) § 1.2). The module is
host-dependent ([`README.md`](README.md) § 5.5): which addresses the host
resolves, whether a bind is permitted, how large a listen backlog the
kernel grants, and how a refused connection reports itself are the host's
affair — the contract below is the shape of every answer. Failures are
ordinary values ([`README.md`](README.md) § 5.1): a dead handle, a
wrong-kind handle, or a refused connection arrives as `Result::Err`, and
end of stream arrives as the `null` inside `string?`.

The qualified path needs no import:

```xulo
std::net::tcp_listen("127.0.0.1:8080")   // ok: Result<number, Error>
std::net::sock_close(1)                  // ok: Result<unit, Error>
```

Import forms are the six of [`README.md`](README.md) § 2.2; the face is the
nine functions — no constants and no types ([`README.md`](README.md) § 3):

```xulo
use std::net::*;                         // the nine functions
use std::net as net;
net::tcp_connect("127.0.0.1:8080")       // ok: same member, aliased
use std::net::{sock_read, sock_write};
```

## Handles

- The three openers — `tcp_listen`, `tcp_connect`, `tcp_accept` — yield the
  handle of a host socket: an opaque **number**, meaningless outside this
  module and accepted only by `std::net` members. Handles are per-process
  names, allocated from `1` upward and **never reused**: after `sock_close`
  the old number stays dead, and a fresh socket always gets a fresh number,
  so a stale handle never names another socket's traffic.
- Version 0.1 has two kinds of handle: a **listener** (`tcp_listen`) and a
  **stream** (`tcp_connect`, and every handle `tcp_accept` yields). A
  member given the wrong kind is `Err`, saying so — never a panic, never
  the operation half applied: `tcp_accept` on a stream is `Err`;
  `sock_read`, `sock_write`, `sock_shutdown`, and `sock_peer_addr` on a
  listener are `Err` (a listener has no peer and no data); `sock_close` and
  `sock_local_addr` accept **either kind**.
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
- Port `0` asks the host for an **ephemeral** port: `tcp_listen` binds one,
  and `sock_local_addr` reads the assigned `"ip:port"` back — a string that
  can be handed straight to `tcp_connect`.

## The openers

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::tcp_listen` | `fn(addr: string): Result<number, Error>` | bind and listen on `addr`; yield the listener's handle |
| `std::net::tcp_connect` | `fn(addr: string): Result<number, Error>` | connect to `addr`; yield the stream's handle |
| `std::net::tcp_accept` | `fn(handle: number): Result<number, Error>` | wait for the next incoming connection on a listener; yield its **new** stream handle |

- `tcp_listen` uses the host's default listen backlog; on success the
  handle names a listener until `sock_close` releases it, and it stays open
  across any number of `tcp_accept` calls. `Err` classes: the address does
  not resolve or is malformed; every resolved address fails to bind —
  address in use, a privileged port, or another host refusal.
- `tcp_connect` **blocks** until the connection completes or fails (a
  blocking member, [`README.md`](README.md) § 5.6). `Err` classes: the
  address does not resolve or is malformed; the connection is refused,
  unreachable, or fails at the host for another reason.
- `tcp_accept`'s operand must be a live listener. It **blocks** until a
  client arrives (§ 5.6) and yields a handle distinct from every other;
  the listener itself stays open and can accept again. `Err` classes: the
  handle is not open; the handle is a stream, not a listener; the host's
  accept fails.

## Reading

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::sock_read` | `fn(handle: number, max: number): Result<string?, Error>` | block for up to `max` bytes of input, decoded as UTF-8 |

- `max` is a byte count in **`1 …= 16777216`** (16 MiB); a fractional value
  counts as its truncation toward zero. A value outside that domain — zero,
  negative, non-finite, or above the cap — is `Err`, checked **before** the
  handle is looked up, so a bad `max` is reported whatever the handle
  names. The domain is an `Err`, never a runtime failure.
- The call **blocks** until at least one byte arrives (§ 5.6) and yields
  **up to** `max` bytes of what has arrived: a short success is ordinary,
  and the next `sock_read` continues the stream where this one stopped.
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
  sticks: every later `sock_read` of that stream keeps yielding `Ok(null)`.
  A host I/O error on the stream is `Err`. Until the peer closes, reading
  is a loop on absence exactly like [`io.md`](io.md)'s line loop: match
  `Ok(null)` as the end, `Ok(chunk)` as data.

## Writing

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::sock_write` | `fn(handle: number, data: string): Result<number, Error>` | write `data`'s UTF-8 bytes; yield the byte count |

- `data` is encoded as UTF-8 and the call **blocks** until every byte is
  handed to the host (§ 5.6): it **loops to completion** and never yields
  a short count the caller must finish. The `Ok` payload is always the
  full UTF-8 byte length of `data` — bytes, not characters, so `"héllo"`
  counts 6. An empty `data` is success with `0`.
- `Err` classes: the handle is not open; the handle is a listener; the
  stream is no longer writable (the peer is gone, or the handle was shut
  down); the host reports an I/O error.

## Shutting down and closing

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::sock_shutdown` | `fn(handle: number): Result<unit, Error>` | half-close a stream in **both** directions |
| `std::net::sock_close` | `fn(handle: number): Result<unit, Error>` | release the handle and close the socket |

- `sock_shutdown` shuts a stream down for reading **and** writing: the peer
  sees end of stream, and this side's reads see it too — a later
  `sock_read` of the handle yields `Ok(null)`. The handle **stays open** in
  the module's registry: a later `sock_close` still succeeds and is what
  actually releases the socket, while a later `sock_write` is `Err` (the
  stream is no longer writable). `Err` classes: the handle is not open; the
  handle is a listener.
- `sock_close` releases the handle and closes the socket it named — a
  listener or a stream. From that moment every member on the number is
  `Err`, `sock_close` itself included: **a second close is `Err`**, not an
  idempotent success. Handles are never reused, so the dead number never
  becomes another socket's name.

## Addresses of a handle

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::net::sock_local_addr` | `fn(handle: number): Result<string, Error>` | the handle's local `"ip:port"` |
| `std::net::sock_peer_addr` | `fn(handle: number): Result<string, Error>` | a stream's remote `"ip:port"` |

- `sock_local_addr` answers for **either kind** of handle: the address the
  listener is bound to, or the stream's local end — including the ephemeral
  port assigned through port `0`, which is how a program learns where to
  connect. `sock_peer_addr` answers only for a **stream**: a listener is
  `Err`, and so is a handle that is not open.
- Both yield the host's `"ip:port"` rendering — bracketed for IPv6 — and
  that string is itself a well-formed argument for `tcp_listen` or
  `tcp_connect`.

## A loopback round trip

Listen on an ephemeral loopback port, read the port back, connect, accept,
then echo in both directions — every call matched, because every call is
fallible:

```xulo
match std::net::tcp_listen("127.0.0.1:0") {
  Result::Ok(server) => match std::net::sock_local_addr(server) {
    Result::Ok(addr) => match std::net::tcp_connect(addr) {
      Result::Ok(client) => match std::net::tcp_accept(server) {
        Result::Ok(accepted) => {
          match std::net::sock_write(client, "ping") {
            Result::Ok(n) => print(n)                    // runs: 4 — bytes
            Result::Err(e) => print(e.message)
          }
          match std::net::sock_read(accepted, 64) {
            Result::Ok(null) => print("eof")
            Result::Ok(text) => print(text)              // runs: ping
            Result::Err(e) => print(e.message)
          }
          match std::net::sock_write(accepted, "pong") {
            Result::Ok(n) => print(n)                    // runs: 4 — bytes
            Result::Err(e) => print(e.message)
          }
          match std::net::sock_read(client, 64) {
            Result::Ok(null) => print("eof")
            Result::Ok(text) => print(text)              // runs: pong
            Result::Err(e) => print(e.message)
          }
          std::net::sock_close(client)                   // ok
          std::net::sock_close(accepted)                 // ok
          std::net::sock_close(server)                   // ok
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
accept — because `tcp_connect` completes through the listen backlog
*before* any `tcp_accept` blocks (see below).

## Small focused cases

On the three handles of a completed handshake — the listener `server`, the
connected `client`, and its accepted peer `accepted`:

```xulo
match std::net::sock_write(client, "héllo") {
  Result::Ok(n) => print(n)              // runs: 6 — the UTF-8 byte count
  Result::Err(e) => print(e.message)
}

std::net::sock_read(accepted, 64)        // runs: Ok("héllo") — buffered data arrives before EOF
std::net::sock_shutdown(client)          // ok: both directions half-closed
std::net::sock_read(accepted, 64)        // runs: Ok(null) — the peer's close is absence, not a failure
std::net::sock_read(accepted, 64)        // runs: Ok(null) — EOF sticks
std::net::sock_read(accepted, 0)         // runs: Err — max outside 1 …= 16777216
std::net::sock_read(999999, 8)           // runs: Err — the handle is not open
std::net::sock_write(server, "x")        // runs: Err — a listener, not a stream

std::net::sock_close(accepted)           // ok: the handle is released
std::net::sock_close(accepted)           // runs: Err — already closed
```

## Blocking and asynchrony

- All nine members are **synchronous** ([`README.md`](README.md) § 5.6): the
  call returns when the host operation completes. The members that can wait
  on the outside world — the openers `tcp_listen`, `tcp_connect`,
  `tcp_accept` and the transfers `sock_read` and `sock_write` — are the
  **blocking members** of the terminology table; while one waits, no other
  task makes progress: the cooperative scheduler is frozen for the
  duration.
- The freeze reaches single-threaded programs too, which makes the
  handshake order load-bearing: `tcp_connect` completes through the listen
  backlog *before* `tcp_accept` is called, so a program that calls
  `tcp_accept` first — with no connection on its way — waits forever.
- Version 0.1 has no asynchronous variant of any member; that is a roadmap
  candidate ([`../README.md`](../README.md) § 1.2), not part of this
  version.

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | every member — `tcp_listen` `tcp_connect` `tcp_accept` `sock_read` `sock_write` `sock_shutdown` `sock_close` `sock_local_addr` `sock_peer_addr` |
| Absence (`Ok(null)`) | `sock_read` at end of stream |
| Total | none |
| Runtime failure | none — `sock_read`'s `max` domain is an `Err`, not a runtime failure |

Failures are ordinary `Result::Err` values carrying `Error { message }`;
message text is informative ([`README.md`](README.md) § 5.1,
[`../README.md`](../README.md) § 3.5), and the contract is the `Err` shape
plus the failure classes each section above enumerates.

## Checked call examples

```xulo
std::net::tcp_listen("127.0.0.1:0")      // ok: Result<number, Error>
std::net::sock_read(1)                   // error[E0207]: expected 2 arguments, found 1
std::net::sock_read("h", 16)             // error[E0202]: expected `number`, found `string`
std::net::open("x")                      // error[E0601]: no member `open`
Net::tcp_listen("127.0.0.1:0")           // error[E0607]: `Net` lives under `std`: use `std::net::tcp_listen`
```
