# `std::http`

`std::http` is the HTTP/1.1 client and server: three functions and three
types that carry requests and responses between programs. It is a protocol
layer over the TCP primitives of [`net.md`](net.md) and re-implements no
transport: every conversation still ends in a socket. The module is
host-dependent ([`README.md`](README.md) § 5.5): which addresses resolve,
how a refusal reports itself, and the reason phrase a host writes are the
host's affair. Version 0.1 is plain `http://` only — HTTPS needs TLS, a
roadmap candidate, so `https://` and every other scheme are `Err`, never a
silent fallback. Failures are ordinary values
([`README.md`](README.md) § 5.1): a malformed URL, a refused connection, an
expired budget, or a protocol violation arrives as `Result::Err`; a header
that is not there arrives as the `null` inside `string?`.

The qualified path needs no import:

```xulo
std::http::fetch("http://example.com/")   // ok: Result<Response, Error>
std::http::listen("127.0.0.1:8080")       // ok: Result<Server, Error>
```

Import forms are the six of [`README.md`](README.md) § 2.2; the face is
three functions and three types ([`README.md`](README.md) § 3):

```xulo
use std::http::*;                         // fetch request listen; Request Response Server
use std::http as http;
http::fetch("http://example.com/")        // ok: same member, aliased
use std::http::{Request, Response, Server};
```

## Types

| Type | Shape | Role |
|------|-------|------|
| `Server` | methods `addr`, `serve` | a bound, listening server |
| `Request` | fields `method`, `path`, `query`, `headers`, `body`, `version`, `remoteAddr`; methods `header`, `queryValue` | one inbound request |
| `Response` | fields `status`, `headers`, `body`; static `text`, `json`; method `header` | one outbound response |

- `Request` and `Response` are plain data: every field is public, and a
  program constructs one with named fields — `Response(status: 200,
  headers: [], body: "ok")`. A `Request` built this way can be handed to a
  handler directly, so handlers are testable without a socket.
- `Server` is opaque: only `listen` issues one, and it has no public
  literal or constructor.
- `headers` and `query` are lists of `{ name: string, value: string }`
  records, in wire order and with duplicates kept. Header lookup is
  case-insensitive, query lookup is exact, and both report absence as
  `null`.

## URLs

- A URL is one string: `http://`, then `host[:port]`, then an optional
  `/path`, then an optional `?query`, then an optional `#fragment`. The
  scheme is matched case-insensitively and `http` is the only accepted
  value; `https` and every other scheme are `Err`.
- `host` is a DNS name, an IPv4 literal, or a bracketed IPv6 literal. The
  port is decimal; when omitted it is `80`, and port `0` is `Err` for a
  client, because a remote port cannot be ephemeral. An empty host is
  `Err`.
- A missing path is `/`. The path and query are sent as written; the
  fragment is not sent. The caller percent-encodes what the URL requires:
  a space, a control character, or a non-ASCII byte in the URL is `Err`.
- The server decodes `%XX` escapes in `Request.path` and in every
  `Request.query` name and value; `+` stays `+`, because form encoding is
  `application/x-www-form-urlencoded` and not URL syntax. A malformed
  escape makes the request malformed (`400`).

## Timeouts

- `fetch` and `request` take an optional trailing `timeout: u64? = null`,
  a budget in milliseconds measured from the call's start and covering
  name resolution, the connection, the request write, and the whole
  response. `null` waits without limit. An expired budget is
  `Result::Err`, a class beside the URL, transport, and protocol classes,
  and it never arrives as absence.
- The server side has no budget in version 0.1: a slow or stalled client
  holds the sequential server for as long as it likes. Per-connection
  deadlines are a roadmap candidate, and so is concurrency.

## Client

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::http::fetch` | `fn(url: string, timeout: u64? = null): Result<Response, Error>` | `GET` and read the whole answer |
| `std::http::request` | `fn(method: string, url: string, headers: List<{ name: string, value: string }>? = null, body: string? = null, timeout: u64? = null): Result<Response, Error>` | any method, with headers and a body |

- `fetch` is exactly `request("GET", url, null, null, timeout)`.
- `method` is sent as given. An empty method, or one containing a space, a
  control character, or a non-ASCII byte, is `Err`.
- `headers` are sent in order with duplicates kept. A name or value
  containing CR, LF, or another control character is `Err`. The client
  adds `Host` from the URL unless the caller supplied one (compared
  case-insensitively), always sends `Connection: close` — the call owns
  the socket for one exchange — and writes `Content-Length` for a
  non-`null` `body`, `0` for an empty string. A caller's `Connection`,
  `Content-Length`, or `Transfer-Encoding` is dropped: framing is always
  the client's.
- The answer's status line carries `HTTP/1.0` or `HTTP/1.1`; another
  version is `Err`. Headers are read up to **64 KiB** and the body up to
  **16 MiB**, both `Err` when exceeded. `Transfer-Encoding: chunked` is
  decoded and its trailers discarded; any other transfer coding is `Err`.
  A `Content-Length` bounds the body; without either, the body runs until
  the connection closes. A status from `100` to `199` is a protocol
  violation (`Err`); `204` and `304` carry no body.
- A response to a `HEAD` request carries no body, whatever its framing
  headers say: the client returns after the headers.
- The body is decoded **lossily** as UTF-8 exactly as
  [`net.md`](net.md)'s `sock_read` decodes it — the client is a byte
  stream dressed as a string. `Content-Encoding` is not decoded and no
  `Accept-Encoding` is sent, so a compressed body arrives as replacement
  characters.
- Redirects are not followed: a `3xx` is returned as-is, `Location`
  header included.
- `Err` classes: the URL is malformed or its scheme is not `http`; the
  connection is refused, unreachable, or times out (the classes of
  [`net.md`](net.md)); the answer violates the protocol; a size cap is
  exceeded.

## Server

| Member | Signature | Meaning |
|--------|-----------|---------|
| `std::http::listen` | `fn(addr: string): Result<Server, Error>` | bind and listen on `addr`; yield the server |
| `Server::addr` | `fn(): Result<string, Error>` | the bound `"ip:port"`, an ephemeral port resolved |
| `Server::serve` | `fn(handler: fn(Request): Response): Result<unit, Error>` | accept and answer requests, one at a time |

- `addr` is the `"host:port"` grammar of [`net.md`](net.md), port `0`
  asking for an ephemeral port. A bind failure is `Err` — address in use,
  a privileged port, or another host refusal — and the backlog is the
  host's default.
- `serve(handler)` runs the whole server: accept one connection, read one
  request, call `handler`, write one response, close. It **blocks** and
  returns only when accepting on the listening socket fails. There is no
  shutdown member in version 0.1: a server runs until the process exits,
  and graceful shutdown is a roadmap candidate that needs concurrency.
- The handler is an ordinary value of type `fn(Request): Response`, so a
  named function or a closure may be passed; it runs on the serving task.
- Version 0.1 is HTTP/1.1. A request line with another version is answered
  `505`, and every connection carries exactly one request — no keep-alive,
  no pipelining.
- Before the handler runs, the request is parsed:
  - the request line is `METHOD SP target SP HTTP/1.1`;
  - the header block — request line and headers together — is at most
    **64 KiB**; a longer one is answered `431`;
  - a header is `name ":" value`, surrounding whitespace trimmed; a
    folded line, a name with no colon, or a control character is malformed
    (`400`); order and duplicates are kept;
  - `Expect: 100-continue`, when a body is expected, is answered with
    the interim `HTTP/1.1 100 Continue` before the body is read; any other
    `Expect` value is answered `417`;
  - a body is read only when `Content-Length` is present and valid: one
    decimal, at most **16 MiB** (larger is `413`, malformed is `400`). A
    `Transfer-Encoding` header is answered `501`, because chunked request
    bodies are not supported. Without `Content-Length` the body is the
    empty string;
  - `path` and `query` are split at the first `?`; without one the query
    is the empty list. The target must start with `/` (`400` otherwise),
    and percent escapes are decoded as [URLs](#urls) describes;
  - `remoteAddr` is the peer's `"ip:port"`.

  The error answers — `400`, `413`, `417`, `431`, `501`, `505` — carry an
  empty body, the status alone, and close the connection; the handler is
  not called. The interim `100` is followed by the final response in the
  usual way.
- A response is written from the handler's value. A status outside
  `200 …= 599` is sent as `500`. The handler's headers are written in
  order except `Content-Length`, `Connection`, and `Transfer-Encoding`,
  which are dropped; the server then writes `Content-Length` — the body's
  UTF-8 byte length — and `Connection: close`. The reason phrase is the
  host's; a client ignores it.
- A `HEAD` request receives the response's headers and `Content-Length`
  but no body. Status `204` and `304` write no body and no
  `Content-Length`.
- A client that disconnects mid-response closes that connection; `serve`
  goes on to the next. Only a failure to accept ends the loop, as `Err`.

## Handlers

Routing and middleware are ordinary Xulo: a handler matches on `method`
and `path`, and a middleware wraps a handler in another function value. No
router is built in; a `Router` would be a library above this module.

```xulo
fn route(req: std::http::Request): std::http::Response {
  if req.method == "GET" && req.path == "/health" {
    std::http::Response.text("ok")
  } else if req.method == "GET" && req.path == "/version" {
    std::http::Response.text("xulo 0.1")
  } else if req.method == "POST" && req.path == "/echo" {
    std::http::Response.text(req.body, 201)
  } else {
    std::http::Response.text("not found", 404)
  }
}

fn logging(
  inner: fn(std::http::Request): std::http::Response
): fn(std::http::Request): std::http::Response {
  (req) => {
    std::io::println(req.method + " " + req.path)
    inner(req)
  }
}

match std::http::listen("127.0.0.1:8080") {
  Result::Ok(server) => match server.serve(logging(route)) {
    Result::Ok(_) => null
    Result::Err(e) => std::io::eprintln(e.message)
  }
  Result::Err(e) => std::io::eprintln(e.message)
}
```

A handler may answer with JSON through `Response.json`, which stringifies
with the rules of [`json.md`](json.md) and sets `Content-Type`:

```xulo
fn status(req: std::http::Request): std::http::Response {
  match std::http::Response.json({ ok: true, path: req.path }) {
    Result::Ok(res) => res
    Result::Err(_) => std::http::Response.text("could not encode", 500)
  }
}
```

`Response.text` sets `Content-Type: text/plain; charset=utf-8` and
`Response.json` sets `Content-Type: application/json`. Both are shorthands
for building the fields yourself: a `Response` you construct writes exactly
the headers you give it.

The client and the server run in separate processes, because `serve`
blocks: one program cannot be both.

```xulo
// The server above is running in another process.
match std::http::fetch("http://127.0.0.1:8080/health", 2000) {
  Result::Ok(res) => {
    print(res.status)                    // runs: 200
    print(res.header("content-type"))    // runs: text/plain; charset=utf-8
    print(res.body)                      // runs: ok
  }
  Result::Err(e) => std::io::eprintln(e.message)
}
```

On the server side, a request's decoded query is data:

```xulo
req.path                     // runs: /search
req.query                    // runs: [{ name: "q", value: "xulo" }, …]
req.queryValue("q")          // runs: xulo
req.header("user-agent")     // runs: null when the client sent none
```

## Blocking and asynchrony

- `fetch`, `request`, and `Server::serve` are **blocking members**
  ([`README.md`](README.md) § 5.6): the first two return when the whole
  exchange completes, `serve` when the listener fails. While one waits,
  the cooperative scheduler is frozen.
- The server therefore answers one request at a time, and a stalled client
  stalls every other client. Concurrency needs `std::async` (roadmap),
  which is why 0.1 has neither keep-alive nor a shutdown member: both are
  meaningful only once requests overlap or can be interrupted.

## Failure model

| Kind | Members |
|------|---------|
| Fallible (`Result`) | `fetch`, `request`, `listen`, `Server::addr`, `Server::serve`, `Response::json` |
| Absence (`string?`) | `Request::header`, `Request::queryValue`, `Response::header` — `null` when absent |
| Total | `Response::text`; construction and field reads of `Request` and `Response` |
| Runtime failure | none — a handler's out-of-range status is sent as `500` |

## Checked call examples

```xulo
std::http::fetch("http://127.0.0.1:8080/health")      // ok: Result<Response, Error>
std::http::fetch()                                    // error[E0207]: expected 1 to 2 arguments, found 0
std::http::fetch("ftp://example.com/")                // ok: Result<Response, Error> — the scheme fails at run time
std::http::request("POST", "http://127.0.0.1:8080/")  // ok: Result<Response, Error>
std::http::request("POST")                            // error[E0207]: expected 2 to 5 arguments, found 1
std::http::request("POST", 1)                         // error[E0202]: expected `string`, found `number`
std::http::listen("127.0.0.1:8080")                   // ok: Result<Server, Error>
std::http::Response(status: 200, headers: [], body: "ok")  // ok: Response
std::http::Response(status: 200)                      // error[E0205]: missing field `headers`
std::http::serve("x", 1)                              // error[E0601]: no member `serve`
Http::fetch("http://127.0.0.1:8080/")                 // error[E0607]: `Http` lives under `std`: use `std::http::fetch`
```
