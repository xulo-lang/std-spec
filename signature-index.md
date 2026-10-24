# Signature Index

Every member of the ten modules in one list, ordered case-insensitively by
member name. Constants show their type; the `std::regex` rows show the type
together with its callable members. The face of each module — the names its
glob binds — is [`modules/README.md`](modules/README.md) § 3; where this
index and a chapter disagree, the chapter governs.

Notation: [`README.md`](README.md) § 3.4 — `T?` optional, `T | U` union,
`fn(A): R` a function, `Task<T>` an asynchronous computation, `Result<T, E>`
the built-in result, `{ f: T, … }` a structural object, `never` a call that
returns no value at all.

| Member | Signature | Module |
|--------|-----------|--------|
| `abs` | `fn(x: number): number` | `std::math` |
| `acos` | `fn(x: float): float` | `std::math` |
| `arch` | `fn(): string` | `std::os` |
| `args` | `fn(): List<string>` | `std::process` |
| `asin` | `fn(x: float): float` | `std::math` |
| `atan` | `fn(x: float): float` | `std::math` |
| `atan2` | `fn(y: float, x: float): float` | `std::math` |
| `basename` | `fn(path: string): string` | `std::path` |
| `cbrt` | `fn(x: float): float` | `std::math` |
| `ceil` | `fn(x: float): int` | `std::math` |
| `chdir` | `fn(path: string): Result<unit, Error>` | `std::os` |
| `cos` | `fn(x: float): float` | `std::math` |
| `countLeadingZeros` | `fn(x: int): int` | `std::math` |
| `countTrailingZeros` | `fn(x: int): int` | `std::math` |
| `cwd` | `fn(): Result<string, Error>` | `std::os` |
| `dirname` | `fn(path: string): string` | `std::path` |
| `E` | `float` (constant) | `std::math` |
| `eprint` | `fn(text: string): Result<unit, Error>` | `std::io` |
| `eprintln` | `fn(text: string): Result<unit, Error>` | `std::io` |
| `exists` | `fn(path: string): bool` | `std::fs` |
| `exit` | `fn(code: number): never` | `std::process` |
| `exp` | `fn(x: float): float` | `std::math` |
| `ext` | `fn(path: string): string` | `std::path` |
| `floor` | `fn(x: float): int` | `std::math` |
| `flush` | `fn(): Result<unit, Error>` | `std::io` |
| `getenv` | `fn(name: string): Result<string?, Error>` | `std::os` |
| `INFINITY` | `float` (constant) | `std::math` |
| `join` | `fn(a: string, b: string): string` | `std::path` |
| `log` | `fn(x: float): float` | `std::math` |
| `log10` | `fn(x: float): float` | `std::math` |
| `log2` | `fn(x: float): float` | `std::math` |
| `max` | `fn(a: number, b: number): number` | `std::math` |
| `mkdir` | `fn(path: string): Result<unit, Error>` | `std::fs` |
| `min` | `fn(a: number, b: number): number` | `std::math` |
| `monotonic` | `fn(): u64` | `std::time` |
| `monotonicNanos` | `fn(): u64` | `std::time` |
| `NAN` | `float` (constant) | `std::math` |
| `normalize` | `fn(path: string): string` | `std::path` |
| `now` | `fn(): u64` | `std::time` |
| `nowNanos` | `fn(): u64` | `std::time` |
| `parse` | `fn(text: string): Result<any, Error>` | `std::json` |
| `PI` | `float` (constant) | `std::math` |
| `pid` | `fn(): number` | `std::os` |
| `platform` | `fn(): string` | `std::os` |
| `popCount` | `fn(x: int): int` | `std::math` |
| `pow` | `fn(base: float, exp: float): float` | `std::math` |
| `print` | `fn(text: string): Result<unit, Error>` | `std::io` (not in the face) |
| `println` | `fn(text: string): Result<unit, Error>` | `std::io` (not in the face) |
| `random` | `fn(): float` | `std::math` |
| `read` | `fn(path: string): Result<string, Error>` | `std::fs` |
| `read` | `fn(): Result<string, Error>` | `std::io` |
| `readLine` | `fn(): Result<string?, Error>` | `std::io` |
| `RegexError` | type `{ message: string }`; constructed only by `RegExp::compile`'s `Err` | `std::regex` |
| `RegexMatch` | type `{ value: string }` with `fn group(index: number): string?` | `std::regex` |
| `RegExp` | type; static `fn compile(pattern: string, flags: string?): Result<RegExp, RegexError>`; methods `fn test(input: string): bool`, `fn find(input: string): RegexMatch?`, `fn find_all(input: string): List<RegexMatch>` | `std::regex` |
| `remove` | `fn(path: string): Result<unit, Error>` | `std::fs` |
| `rotateLeft` | `fn(x: int, amount: u64): int` | `std::math` |
| `rotateRight` | `fn(x: int, amount: u64): int` | `std::math` |
| `round` | `fn(x: float): int` | `std::math` |
| `run` | `fn(cmd: string, input: string? = null): Result<{ stdout: string, stderr: string, code: number }, Error>` | `std::process` |
| `setenv` | `fn(name: string, value: string): Result<unit, Error>` | `std::os` |
| `sin` | `fn(x: float): float` | `std::math` |
| `sleep` | `fn(ms: u64): Task<unit>` | `std::time` |
| `sock_close` | `fn(handle: number): Result<unit, Error>` | `std::net` |
| `sock_local_addr` | `fn(handle: number): Result<string, Error>` | `std::net` |
| `sock_peer_addr` | `fn(handle: number): Result<string, Error>` | `std::net` |
| `sock_read` | `fn(handle: number, max: number): Result<string?, Error>` | `std::net` |
| `sock_shutdown` | `fn(handle: number): Result<unit, Error>` | `std::net` |
| `sock_write` | `fn(handle: number, data: string): Result<number, Error>` | `std::net` |
| `sqrt` | `fn(x: float): float` | `std::math` |
| `string.replace` | `fn(pattern: RegExp, replacer: string \| fn(RegexMatch): string): string` | `std::regex` |
| `stringify` | `fn(value: any): Result<string, Error>` | `std::json` |
| `tan` | `fn(x: float): float` | `std::math` |
| `TAU` | `float` (constant) | `std::math` |
| `tcp_accept` | `fn(handle: number): Result<number, Error>` | `std::net` |
| `tcp_connect` | `fn(addr: string): Result<number, Error>` | `std::net` |
| `tcp_listen` | `fn(addr: string): Result<number, Error>` | `std::net` |
| `toFixed` | `fn(x: float, digits: int): string` | `std::math` |
| `trunc` | `fn(x: float): int` | `std::math` |
| `wrappingAdd` | `fn(a: int, b: int): int` | `std::math` |
| `wrappingMul` | `fn(a: int, b: int): int` | `std::math` |
| `wrappingSub` | `fn(a: int, b: int): int` | `std::math` |
| `write` | `fn(path: string, contents: string): Result<unit, Error>` | `std::fs` |
