# Signature Index

Every member of the eleven modules in one list, ordered case-insensitively by
member name. Constants show their type; a type's row shows it together with its
callable members. The face of each module — the names its
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
| `append` | `fn(path: string, contents: string): Result<unit, Error>` | `std::fs` |
| `appendBytes` | `fn(path: string, bytes: List<u8>): Result<unit, Error>` | `std::fs` |
| `arch` | `fn(): string` | `std::os` |
| `args` | `fn(): List<string>` | `std::process` |
| `asin` | `fn(x: float): float` | `std::math` |
| `atan` | `fn(x: float): float` | `std::math` |
| `atan2` | `fn(y: float, x: float): float` | `std::math` |
| `basename` | `fn(path: string): string` | `std::path` |
| `cbrt` | `fn(x: float): float` | `std::math` |
| `ceil` | `fn(x: float): int` | `std::math` |
| `chdir` | `fn(path: string): Result<unit, Error>` | `std::os` |
| `Child` | type; methods `fn wait(): Result<{ stdout: string, stderr: string, code: number }, Error>`, `fn kill(): Result<unit, Error>`, `fn pid(): number` | `std::process` |
| `clamp` | `fn(x: number, lo: number, hi: number): number` | `std::math` |
| `Command` | type; static `fn shell(cmd: string): Command`, `fn direct(program: string, args: List<string>): Command`; setters `fn input(s: string): Command`, `fn cwd(path: string): Command`, `fn env(name: string, value: string): Command`, `fn timeout(ms: u64): Command`; methods `fn run(): Result<{ stdout: string, stderr: string, code: number }, Error>`, `fn launch(): Result<Child, Error>` | `std::process` |
| `copy` | `fn(from: string, to: string): Result<unit, Error>` | `std::fs` |
| `cos` | `fn(x: float): float` | `std::math` |
| `countLeadingZeros` | `fn(x: int): int` | `std::math` |
| `countTrailingZeros` | `fn(x: int): int` | `std::math` |
| `cpuCount` | `fn(): number` | `std::os` |
| `createNew` | `fn(path: string, contents: string): Result<unit, Error>` | `std::fs` |
| `cwd` | `fn(): Result<string, Error>` | `std::os` |
| `dirname` | `fn(path: string): string` | `std::path` |
| `E` | `float` (constant) | `std::math` |
| `eprint` | `fn(text: string): Result<unit, Error>` | `std::io` |
| `eprintln` | `fn(text: string): Result<unit, Error>` | `std::io` |
| `exec` | `fn(program: string, args: List<string>, input: string? = null, timeout: u64? = null): Result<{ stdout: string, stderr: string, code: number }, Error>` | `std::process` |
| `execPath` | `fn(): Result<string, Error>` | `std::os` |
| `exists` | `fn(path: string): bool` | `std::fs` |
| `exit` | `fn(code: number): never` | `std::process` |
| `exp` | `fn(x: float): float` | `std::math` |
| `expm1` | `fn(x: float): float` | `std::math` |
| `ext` | `fn(path: string): string` | `std::path` |
| `fetch` | `fn(url: string, timeout: u64? = null): Result<Response, Error>` | `std::http` |
| `floor` | `fn(x: float): int` | `std::math` |
| `flush` | `fn(): Result<unit, Error>` | `std::io` |
| `format` | `fn(ms: u64): string` | `std::time` |
| `fromParts` | `fn(year: int, month: int, day: int, hour: int = 0, minute: int = 0, second: int = 0, millisecond: int = 0): u64` | `std::time` |
| `getenv` | `fn(name: string): Result<string?, Error>` | `std::os` |
| `homeDir` | `fn(): Result<string, Error>` | `std::os` |
| `hostName` | `fn(): Result<string, Error>` | `std::os` |
| `hypot` | `fn(a: float, b: float): float` | `std::math` |
| `INFINITY` | `float` (constant) | `std::math` |
| `isAbsolute` | `fn(path: string): bool` | `std::path` |
| `join` | `fn(a: string, b: string): string` | `std::path` |
| `launch` | `fn(cmd: string, input: string? = null, timeout: u64? = null): Result<Child, Error>` | `std::process` |
| `listen` | `fn(addr: string): Result<Server, Error>` | `std::http` |
| `log` | `fn(x: float): float` | `std::math` |
| `log10` | `fn(x: float): float` | `std::math` |
| `log1p` | `fn(x: float): float` | `std::math` |
| `log2` | `fn(x: float): float` | `std::math` |
| `max` | `fn(a: number, b: number): number` | `std::math` |
| `min` | `fn(a: number, b: number): number` | `std::math` |
| `mkdir` | `fn(path: string): Result<unit, Error>` | `std::fs` |
| `monotonic` | `fn(): u64` | `std::time` |
| `monotonicNanos` | `fn(): u64` | `std::time` |
| `NAN` | `float` (constant) | `std::math` |
| `normalize` | `fn(path: string): string` | `std::path` |
| `now` | `fn(): u64` | `std::time` |
| `nowNanos` | `fn(): u64` | `std::time` |
| `parse` | `fn(text: string): Result<any, Error>` | `std::json` |
| `parse` | `fn(text: string): Result<u64, Error>` | `std::time` |
| `parts` | `fn(ms: u64): { year: int, month: int, day: int, hour: int, minute: int, second: int, millisecond: int }` | `std::time` |
| `PI` | `float` (constant) | `std::math` |
| `pid` | `fn(): number` | `std::os` |
| `platform` | `fn(): string` | `std::os` |
| `popCount` | `fn(x: int): int` | `std::math` |
| `pow` | `fn(base: float, exp: float): float` | `std::math` |
| `print` | `fn(text: string): Result<unit, Error>` | `std::io` (not in the face) |
| `println` | `fn(text: string): Result<unit, Error>` | `std::io` (not in the face) |
| `random` | `fn(): float` | `std::math` |
| `randomInt` | `fn(lo: int, hi: int): int` | `std::math` |
| `read` | `fn(path: string): Result<string, Error>` | `std::fs` |
| `read` | `fn(): Result<string, Error>` | `std::io` |
| `readBytes` | `fn(path: string): Result<List<u8>, Error>` | `std::fs` |
| `readDir` | `fn(path: string): Result<List<string>, Error>` | `std::fs` |
| `readLine` | `fn(): Result<string?, Error>` | `std::io` |
| `readN` | `fn(max: number): Result<string?, Error>` | `std::io` |
| `realpath` | `fn(path: string): Result<string, Error>` | `std::fs` |
| `RegexError` | type `{ message: string }`; constructed only by `RegExp::compile`'s `Err` | `std::regex` |
| `RegexMatch` | type `{ value: string, index: int, end: int }` with `fn group(ref: number \| string): string?` | `std::regex` |
| `RegExp` | type; static `fn compile(pattern: string, flags: string?): Result<RegExp, RegexError>`; methods `fn test(input: string): bool`, `fn find(input: string): RegexMatch?`, `fn findAll(input: string): List<RegexMatch>` | `std::regex` |
| `relative` | `fn(from: string, to: string): string?` | `std::path` |
| `remove` | `fn(path: string): Result<unit, Error>` | `std::fs` |
| `removeDir` | `fn(path: string): Result<unit, Error>` | `std::fs` |
| `rename` | `fn(from: string, to: string): Result<unit, Error>` | `std::fs` |
| `Request` | type; fields `method: string`, `path: string`, `query: List<{ name: string, value: string }>`, `form: List<{ name: string, value: string }>`, `headers: List<{ name: string, value: string }>`, `body: string`, `version: string`, `remoteAddr: string`; methods `fn header(name: string): string?`, `fn queryValue(name: string): string?`, `fn formValue(name: string): string?` | `std::http` |
| `request` | `fn(method: string, url: string, headers: List<{ name: string, value: string }>? = null, body: string? = null, timeout: u64? = null): Result<Response, Error>` | `std::http` |
| `Response` | type; fields `status: number`, `headers: List<{ name: string, value: string }>`, `body: string`; static `fn text(body: string, status: number = 200): Response`, `fn json(value: any, status: number = 200): Result<Response, Error>`; method `fn header(name: string): string?` | `std::http` |
| `rotateLeft` | `fn(x: int, amount: u64): int` | `std::math` |
| `rotateRight` | `fn(x: int, amount: u64): int` | `std::math` |
| `round` | `fn(x: float): int` | `std::math` |
| `run` | `fn(cmd: string, input: string? = null, timeout: u64? = null): Result<{ stdout: string, stderr: string, code: number }, Error>` | `std::process` |
| `Server` | type; methods `fn addr(): Result<string, Error>`, `fn serve(handler: fn(Request): Response): Result<unit, Error>`; constructed only by `listen` | `std::http` |
| `setenv` | `fn(name: string, value: string): Result<unit, Error>` | `std::os` |
| `setMode` | `fn(path: string, mode: number): Result<unit, Error>` | `std::fs` |
| `setTimes` | `fn(path: string, modified: u64): Result<unit, Error>` | `std::fs` |
| `sign` | `fn(x: number): number` | `std::math` |
| `sin` | `fn(x: float): float` | `std::math` |
| `sleep` | `fn(ms: u64): Task<unit>` | `std::time` |
| `sockClose` | `fn(handle: number): Result<unit, Error>` | `std::net` |
| `sockLocalAddr` | `fn(handle: number): Result<string, Error>` | `std::net` |
| `sockPeerAddr` | `fn(handle: number): Result<string, Error>` | `std::net` |
| `sockRead` | `fn(handle: number, max: number, timeout: u64? = null): Result<string?, Error>` | `std::net` |
| `sockShutdown` | `fn(handle: number): Result<unit, Error>` | `std::net` |
| `sockShutdownWrite` | `fn(handle: number): Result<unit, Error>` | `std::net` |
| `sockWrite` | `fn(handle: number, data: string, timeout: u64? = null): Result<number, Error>` | `std::net` |
| `split` | `fn(path: string): { dir: string, base: string, ext: string }` | `std::path` |
| `sqrt` | `fn(x: float): float` | `std::math` |
| `stat` | `fn(path: string): Result<{ kind: string, size: number, modified: u64, mode: number }, Error>` | `std::fs` |
| `stdinIsTTY` | `fn(): bool` | `std::io` |
| `stdoutIsTTY` | `fn(): bool` | `std::io` |
| `stem` | `fn(path: string): string` | `std::path` |
| `string.replace` | `fn(pattern: RegExp, replacer: string \| fn(RegexMatch): string): string` | `std::regex` |
| `string.split` | `fn(re: RegExp): List<string>` | `std::regex` |
| `stringify` | `fn(value: any, indent: number? = null): Result<string, Error>` | `std::json` |
| `tan` | `fn(x: float): float` | `std::math` |
| `TAU` | `float` (constant) | `std::math` |
| `tcpAccept` | `fn(handle: number, timeout: u64? = null): Result<number, Error>` | `std::net` |
| `tcpConnect` | `fn(addr: string, timeout: u64? = null): Result<number, Error>` | `std::net` |
| `tcpListen` | `fn(addr: string): Result<number, Error>` | `std::net` |
| `tempDir` | `fn(prefix: string? = null): Result<string, Error>` | `std::fs` |
| `tempDirIn` | `fn(dir: string, prefix: string? = null): Result<string, Error>` | `std::fs` |
| `tempFile` | `fn(prefix: string? = null): Result<string, Error>` | `std::fs` |
| `tempFileIn` | `fn(dir: string, prefix: string? = null): Result<string, Error>` | `std::fs` |
| `tmpDir` | `fn(): string` | `std::os` |
| `toFixed` | `fn(x: float, digits: int): string` | `std::math` |
| `touch` | `fn(path: string): Result<unit, Error>` | `std::fs` |
| `trunc` | `fn(x: float): int` | `std::math` |
| `unsetenv` | `fn(name: string): Result<unit, Error>` | `std::os` |
| `userName` | `fn(): Result<string, Error>` | `std::os` |
| `valid` | `fn(text: string): bool` | `std::json` |
| `walk` | `fn(path: string): Result<List<string>, Error>` | `std::fs` |
| `weekday` | `fn(ms: u64): int` | `std::time` |
| `withExt` | `fn(path: string, ext: string): string` | `std::path` |
| `wrappingAdd` | `fn(a: int, b: int): int` | `std::math` |
| `wrappingMul` | `fn(a: int, b: int): int` | `std::math` |
| `wrappingSub` | `fn(a: int, b: int): int` | `std::math` |
| `write` | `fn(path: string, contents: string): Result<unit, Error>` | `std::fs` |
| `writeBytes` | `fn(path: string, bytes: List<u8>): Result<unit, Error>` | `std::fs` |
