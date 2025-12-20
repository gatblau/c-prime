# Component spec — cprime-host-crypto, cprime-host-clock, cprime-host-random, cprime-host-log

> Spec set: cprime
> Status: Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> Last updated: 2025-12-20

### SPEC: cprime-host-providers
**File:** `providers/cprime-host-crypto/src/lib.rs`, `providers/cprime-host-clock/src/lib.rs`, `providers/cprime-host-random/src/lib.rs`, `providers/cprime-host-log/src/lib.rs` | **Package:** `cprime-host-crypto`, `cprime-host-clock`, `cprime-host-random`, `cprime-host-log` (each version `0.1.0`) | **Phase:** 3 | **Dependencies:** `cprime-provider` (workspace); third-party: `sha2`, `hmac` (crypto), `blake3` (crypto), `getrandom` (random)

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

The four built-in providers implement the host capabilities that ship with C′ (ADR-020): hashing and MAC (`host::Crypto`), wall-clock and monotonic time (`host::Clock`), operating-system randomness (`host::Random`) and program logging (`host::Log`). None offers network or filesystem access. Each is an ordinary provider crate written on the `cprime-provider` SDK, so the built-ins exercise the same mechanism third-party providers use, and each is linked into every project runtime without a `cprime.toml` entry (SA-09).

#### Approach

One crate per provider follows the rule "one capability per provider" and lets the provider inventory list each built-in with its own version. Every operation is a thin `OpFn` wrapping a well-audited crate inside `cprime_provider::sdk::guard`, so a library panic becomes a `ProviderPanic` trap instead of unwinding into compiled code. Rejected: one `cprime-host` crate exposing four providers — it would break the one-capability-per-crate rule and give all four one inventory line. Rejected: implementing SHA-256 or HMAC in-house — the TCB argument favours the RustCrypto and BLAKE3 reference crates. `host::Log` hand-writes its JSON line (fixed four fields, explicit escaping) instead of depending on `serde_json`, so the crate depends on `cprime-provider` alone.

#### Shared Context

Copied from `phase-2-architecture.md` §2C and §2E.

```rust
#[derive(Clone, PartialEq, Eq)]
pub enum BoundaryType { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<BoundaryType>), Result(Box<BoundaryType>, Box<BoundaryType>) }

pub type OpFn = unsafe extern "C" fn(args: *const u64, rt: *mut RtContext) -> u64;

#[derive(Clone)] pub struct OpDecl { pub name: &'static str, pub params: Vec<BoundaryType>, pub ret: BoundaryType, pub deterministic: bool, pub func: OpFn,
                                    pub vectors: &'static [KnownAnswer] }   // (D40)
#[derive(Clone, Copy)] pub struct KnownAnswer { pub args: &'static [Lit], pub ret: Lit }
#[derive(Clone, Copy, PartialEq, Eq)] pub enum Lit { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(&'static [u8]) }
// deterministic (ADR-054): equal results for equal arguments, no observable effect, returns without waiting on anything outside the call
#[derive(Clone)] pub struct ProviderDecl { pub path: &'static str, pub crate_name: &'static str, pub crate_version: &'static str, pub ops: Vec<OpDecl>, pub thread_safe: bool, pub transport: Option<TransportDecl> }

#[repr(C)]
pub struct RtContext { pub trap: u32, pub mode: u32, pub fuel: u64, pub depth: u64, pub max_depth: u64, pub mem_limit: u64, pub mem_live: u64, pub tracker: *mut AllocTracker,
                       pub site: u64, pub chain: *mut u32 }  // field order is ABI; site (offset 56) and chain (offset 64) are RFC 03 additions (D27, D28)

#[repr(u32)]
pub enum RtTrap { None = 0, Overflow = 1, DivisionByZero = 2, IndexOutOfBounds = 3, Unreachable = 4, FuelExhausted = 5, MemoryLimit = 6, ProviderPanic = 7, StackLimit = 8 }
```

This crate never reads or writes `site` or `chain`; the release runtime uses them to name a panicking operation and to print the call chain (`components/cprime-provider.md` step 8). Every `RtContext` this crate's tests build comes from `RtContext::sandbox`, or from a struct literal that sets `site: 0` and `chain: std::ptr::null_mut()`. Built-in provider behaviour is unchanged by RFC 03.

SDK helpers used (from `components/cprime-provider.md`): `arg_u64`, `arg_bytes`, `ret_unit`, `ret_u64`, `ret_bytes`, `guard`, `cprime_rt_trap`, trait `Provider { fn decl() -> ProviderDecl; }`, and the crate-root convention `pub fn cprime_provider() -> ProviderDecl`. Ownership: arguments are borrowed and valid only during the call; results are owned. Value representation: one 64-bit word per value; `Unit` = 0; `Bytes` and `FixedBytes[N]` are heap cells of kind 0.

Built-in operations (SA-35, normative):

| Provider | Operation | C′ signature | `deterministic` |
|---|---|---|---|
| `host::Crypto` | `sha256` | `(msg Bytes) FixedBytes[32]` | yes |
| `host::Crypto` | `hmac_sha256` | `(key Bytes, msg Bytes) FixedBytes[32]` | yes |
| `host::Crypto` | `blake3` | `(msg Bytes) FixedBytes[32]` | yes |
| `host::Clock` | `now_unix_seconds` | `() U64` | no |
| `host::Clock` | `monotonic_nanos` | `() U64` | no |
| `host::Random` | `random_bytes` | `(n U64) Bytes` | no |
| `host::Random` | `random_u64` | `() U64` | no |
| `host::Log` | `info` | `(msg Bytes)` | no |
| `host::Log` | `warn` | `(msg Bytes)` | no |
| `host::Log` | `error` | `(msg Bytes)` | no |

Fuzzing behaviour (SA-30): during T3, `cprime-verify` calls the real provider only for `deterministic` operations of a bound capability; `host::Clock`, `host::Random` and `host::Log` operations are replaced by stubs, so fuzzing never reads the clock, draws OS randomness or writes log lines. The double-call determinism probe calls each bound `host::Crypto` operation twice with identical inputs.

#### Public Interface

Each crate exposes exactly two public items: the provider type and the root function.

```rust
// cprime-host-crypto
pub struct Crypto;
impl cprime_provider::sdk::Provider for Crypto { fn decl() -> ProviderDecl; }
pub fn cprime_provider() -> ProviderDecl;     // == Crypto::decl()

// cprime-host-clock
pub struct Clock;   impl Provider for Clock  { fn decl() -> ProviderDecl; }   pub fn cprime_provider() -> ProviderDecl;
// cprime-host-random
pub struct Random;  impl Provider for Random { fn decl() -> ProviderDecl; }   pub fn cprime_provider() -> ProviderDecl;
// cprime-host-log
pub struct Log;     impl Provider for Log    { fn decl() -> ProviderDecl; }   pub fn cprime_provider() -> ProviderDecl;
```

`thread_safe` (ADR-045, D21, amended by ADR-051 and SA-69): all four built-ins declare `true`. `host::Crypto` keeps no state. `host::Clock` keeps its monotonic base instant in a `OnceLock`, set by the first call from any thread. `host::Random` draws from a per-thread generator, seeded from the operating system on first use in each thread. `host::Log` writes each line with one `write` call while holding the stderr lock, so lines from different threads never interleave.

Declarations returned (operation order is normative; it fixes the global operation indices 0–9 because built-ins come first in every registry):

| Crate | `path` | `crate_name` | `crate_version` | `ops` (flat index) |
|---|---|---|---|---|
| `cprime-host-crypto` | `host::Crypto` | `cprime-host-crypto` | `0.1.0` | `sha256` (0), `hmac_sha256` (1), `blake3` (2) |
| `cprime-host-clock` | `host::Clock` | `cprime-host-clock` | `0.1.0` | `now_unix_seconds` (3), `monotonic_nanos` (4) |
| `cprime-host-random` | `host::Random` | `cprime-host-random` | `0.1.0` | `random_bytes` (5), `random_u64` (6) |
| `cprime-host-log` | `host::Log` | `cprime-host-log` | `0.1.0` | `info` (7), `warn` (8), `error` (9) |

`params`/`ret` per operation use `BoundaryType` exactly as the table in *Shared Context* (`FixedBytes[32]` = `BoundaryType::FixedBytes(32)`). `crate_name` and `crate_version` are taken from `env!("CARGO_PKG_NAME")` and `env!("CARGO_PKG_VERSION")`.

**Known-answer vectors (D40, ADR-054, SA-74).** Each `deterministic` operation declares `vectors` as a `static` slice; every non-deterministic operation declares `vectors: &[]`. The byte values of every vector are copied verbatim from the cited source when the crate is written; none is computed by the crate's own code.

| Operation | Vectors, in order | Source |
|---|---|---|
| `sha256` | (0) `msg` = empty; (1) `msg` = `"abc"` (`0x616263`), `ret` = `0xba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad` | FIPS 180-4 examples (NIST CSRC "SHA-256" example document; empty-message digest from NIST CAVP `SHA256ShortMsg`, `Len = 0`) |
| `hmac_sha256` | test cases 1, 2, 3, 4, 6 and 7, each with its key, data and the `HMAC-SHA-256` result | RFC 4231 §4.2, §4.3, §4.4, §4.5, §4.7, §4.8; test case 5 (§4.6) truncates the output to 128 bits and is not used |
| `blake3` | input lengths 0 and 1 of the official vectors (input byte *i* = *i* mod 251), `ret` = the first 32 bytes of the published `hash` | BLAKE3 reference repository, `test_vectors/test_vectors.json` |

`sdk::check_vectors` (`cprime-provider` step 17) runs them through `cprimed --check-providers` during every `cprimec runtime build`; a mismatch refuses the runtime.

##### Example

`.cprime` binding and an `.opq` use, matching the concept example:

```text
// auth.cprime (module Auth, also declares the entry point)
capability CryptoCap {
    deterministic hmac_sha256(key, msg Bytes) Hash
}
capability LogCap {
    info(msg Bytes)
}
bind CryptoCap = host::Crypto
bind LogCap = host::Log
```

Inputs and outputs:

| Call | Input | Output |
|---|---|---|
| `host::Crypto::sha256` | `msg = 0x616263` (`"abc"`) | `0xba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad` |
| `host::Crypto::hmac_sha256` | RFC 4231 test case 1 (key = twenty `0x0b` bytes, msg = `"Hi There"`) | the HMAC-SHA-256 value published in RFC 4231 §4.2 |
| `host::Crypto::blake3` | `msg = 0x` (empty) | the empty-input hash published in the BLAKE3 reference test vectors (`test_vectors.json`, input length 0) |
| `host::Random::random_bytes` | `n = 4` | 4 bytes from the OS source, e.g. `0x9c1f07e2` |
| `host::Log::info` | `msg = 0x68656c6c6f` (`"hello"`) | `()`; stderr line `{"ts":"2025-12-20T10:15:30.123Z","level":"info","target":"program","msg":"hello"}` |

Operation body shape (normative pattern for all ten operations):

```rust
unsafe extern "C" fn hmac_sha256(args: *const u64, rt: *mut RtContext) -> u64 {
    guard(rt, || {
        let key = arg_bytes(args, 0);
        let msg = arg_bytes(args, 1);
        let mut mac = <hmac::Hmac<sha2::Sha256> as hmac::Mac>::new_from_slice(key)
            .expect("HMAC accepts keys of any length");
        hmac::Mac::update(&mut mac, msg);
        let tag = hmac::Mac::finalize(mac).into_bytes();   // 32 bytes
        ret_bytes(rt, &tag)
    })
}
```

#### Internal Logic

Every operation runs its body inside `guard(rt, …)`; arguments are read with `arg_*` and never retained; results are built with `ret_*`. No operation logs through the C′ logging cross-cutting facility; only `host::Log` writes, and only its own program lines.

1. **`host::Crypto::sha256(msg)`** — read `msg` with `arg_bytes(args, 0)`; compute `sha2::Sha256::digest(msg)` (32 bytes); return `ret_bytes(rt, &digest)`. Errors: allocation refusal → `MemoryLimit` (recorded by `ret_bytes`, result 0). Deterministic: output depends only on `msg`.
2. **`host::Crypto::hmac_sha256(key, msg)`** — HMAC-SHA-256 per RFC 2104 with `hmac::Hmac<sha2::Sha256>`; any key length, including empty, is accepted (keys longer than 64 bytes are hashed first, per RFC 2104); return the 32-byte tag. Errors: `new_from_slice` cannot fail for HMAC; the `expect` is reached only by a library defect and becomes `ProviderPanic`; allocation refusal → `MemoryLimit`.
3. **`host::Crypto::blake3(msg)`** — `blake3::hash(msg).as_bytes()` (32 bytes, default unkeyed mode); return via `ret_bytes`. Errors: `MemoryLimit` only.
4. **`host::Clock::now_unix_seconds()`** — `SystemTime::now().duration_since(UNIX_EPOCH)`; `Ok(d)` → `d.as_secs()`; `Err(_)` (clock set before 1970-01-01) → 0. Return `ret_u64`. No error path.
5. **`host::Clock::monotonic_nanos()`** — a `static ANCHOR: OnceLock<Instant>` is initialised on the first call of this operation in the running program; return `ANCHOR.elapsed().as_nanos()` saturated to `u64::MAX`. The first call returns a value ≤ the time taken by the call itself; values never decrease within one program run. No error path.
6. **`host::Random::random_bytes(n)`** — `n = arg_u64(args, 0)`. If `n > 1_048_576` (1 MiB): call `cprime_rt_trap(rt, MemoryLimit)` and return 0. Otherwise fill a `Vec<u8>` of length `n` with `getrandom::getrandom`; return `ret_bytes`. Errors: OS randomness failure → the `expect("OS randomness source unavailable")` panics → `ProviderPanic`; allocation refusal → `MemoryLimit`. `n = 0` returns empty `Bytes` without calling `getrandom`.
7. **`host::Random::random_u64()`** — fill 8 bytes with `getrandom::getrandom`, return `u64::from_le_bytes`. Errors: OS failure → `ProviderPanic`.
8. **`host::Log::info|warn|error(msg)`** — (a) `level` is the constant `"info"`, `"warn"` or `"error"`. (b) Timestamp: `SystemTime::now()` since `UNIX_EPOCH` (pre-epoch → 0), formatted as RFC 3339 UTC with millisecond precision `YYYY-MM-DDTHH:MM:SS.mmmZ`, computed with the days-to-civil-date algorithm (no date crate). (c) Message: `String::from_utf8_lossy(msg)`, JSON-escaped: `"` → `\"`, `\` → `\\`, U+0000–U+001F → `\u00XX` (lower-case hex), all other characters written as UTF-8. (d) Build the line `{"ts":"<ts>","level":"<level>","target":"program","msg":"<escaped>"}` plus `\n` in one `String`. (e) Write it with one `write_all` on a locked `std::io::stderr()`; a write error is ignored (logging never traps a program). (f) Return `ret_unit()`.

#### Data Model

N/A — no persistent storage. In-memory state: the `monotonic_nanos` anchor (`OnceLock<Instant>`, one per program run). All other operations are stateless.

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Result allocation refused (memory limit reached) in any operation returning `Bytes` or `FixedBytes[32]` | mode 1: trap recorded, word 0; release: exit 5 | `RtTrap::MemoryLimit` (6) | release stderr: `cprime: memory limit exceeded` |
| `random_bytes` called with `n > 1,048,576` | mode 1: trap recorded, word 0; release: exit 5 | `RtTrap::MemoryLimit` (6) | release stderr: `cprime: memory limit exceeded` |
| OS randomness source fails (`getrandom` returns `Err`) | mode 1: trap recorded, word 0; release: exit 6 | `RtTrap::ProviderPanic` (7) | panic message `OS randomness source unavailable: <error>`, then `cprime: provider panic` |
| Any library panic inside an operation body | mode 1: trap recorded; release: exit 6 | `RtTrap::ProviderPanic` (7) | panic message, then `cprime: provider panic` |
| System clock earlier than 1970-01-01 | normal return | — | `now_unix_seconds` returns 0; log `ts` is `1970-01-01T00:00:00.000Z` |
| Standard error closed or write fails in `host::Log` | normal return | — | line dropped; operation returns `()` |

#### Acceptance Criteria (Gherkin)

Feature: cprime-host-providers

  Scenario: Happy path — SHA-256 known answer
    Given the host::Crypto provider in a sandbox context
    When sha256 is called with Bytes 0x616263
    Then the result is FixedBytes[32] 0xba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad

  Scenario: Happy path — HMAC-SHA-256 matches RFC 4231 test case 1
    Given key = twenty 0x0b bytes and msg = "Hi There"
    When hmac_sha256 is called
    Then the 32-byte result equals the HMAC-SHA-256 value of RFC 4231 §4.2 test case 1

  Scenario: Happy path — BLAKE3 of empty input matches the reference vector
    When blake3 is called with empty Bytes
    Then the result equals the BLAKE3 reference test vector for input length 0

  Scenario: Happy path — every built-in vector passes (RFC 07 criterion 7)
    Given the registry of the four built-in providers
    When cprime_provider::sdk::check_vectors runs
    Then it returns no failure
    And sha256 declares 2 vectors, hmac_sha256 6 and blake3 2, and every other built-in operation declares 0

  Scenario: Error path — a corrupted vector is reported (RFC 07 criterion 7)
    Given a test copy of host::Crypto whose hmac_sha256 vector 0 has its last result byte flipped
    When check_vectors runs
    Then it returns one VectorFailure for host::Crypto operation hmac_sha256 index 0 with a detail starting "result differs"

  Scenario: Happy path — declarations validate and fix flat indices 0 to 9
    Given a registry of Crypto, Clock, Random and Log in that order
    When validate_registry is called
    Then it returns Ok
    And flat_ops lists sha256 at index 0 and error at index 9
    And exactly sha256, hmac_sha256 and blake3 have deterministic = true

  Scenario: Edge case — determinism probe passes for every crypto operation
    Given msg = 0x00ff and key = 0x01
    When each host::Crypto operation is called twice with the same arguments
    Then the two results of each operation are equal byte for byte

  Scenario: Edge case — random_bytes with n = 0
    When random_bytes is called with n = 0
    Then the result is empty Bytes and no trap is recorded

  Scenario: Edge case — monotonic_nanos never decreases
    When monotonic_nanos is called 1,000 times in sequence
    Then each value is greater than or equal to the previous one

  Scenario: Edge case — log line escapes control characters and invalid UTF-8
    Given msg = 0x61220a62ff
    When host::Log::info is called with stderr captured
    Then exactly one line is written
    And its msg field is "a\"\u000ab�"
    And level is "info" and target is "program"

  Scenario: Error path — random_bytes above 1 MiB traps MemoryLimit
    Given a sandbox context
    When random_bytes is called with n = 1048577
    Then the result is 0 and rt.trap is MemoryLimit (6)

  Scenario: Error path — memory limit reached while returning a digest
    Given a sandbox context whose mem_limit leaves fewer than 56 bytes free
    When sha256 is called
    Then the result is 0 and rt.trap is MemoryLimit (6)

  Scenario: Happy path — every built-in is thread-safe
    Given the four built-in provider declarations
    When their thread_safe flags are read
    Then all four declare thread_safe = true

  Scenario: Edge case — concurrent calls stay well-formed
    Given 16 threads, each with its own RtContext, calling info, random_u64 and monotonic_nanos 10,000 times
    When the calls complete
    Then every stderr line is a complete JSON object, every thread's monotonic_nanos values never decrease, and no call panics

#### Performance, Security, Observability

- **Performance targets:** `sha256`, `hmac_sha256`, `blake3` add ≤ 2 µs p95 over the underlying library for inputs ≤ 1 KiB (allocation and copy only); `now_unix_seconds`, `monotonic_nanos`, `random_u64` ≤ 2 µs p95; `random_bytes` bounded at 1 MiB per call; each `host::Log` call performs exactly one `write` system call for lines ≤ 64 KiB.
- **Security:** all four crates join the TCB and appear in every provider inventory. No network or filesystem access. `host::Crypto` operations are the only `deterministic` built-ins; their determinism is checked by the double-call probe during fuzzing. Randomness comes only from the OS source through `getrandom`; there is no seeded fallback. `host::Log` writes program-supplied bytes to stderr only, escaped so a message cannot inject a second JSON line. No timing side-channel guarantees (HLD non-goal); `hmac_sha256` is not intended for constant-time tag comparison, which is a program concern.
- **Observability:** `host::Log` lines use fields `ts`, `level`, `target` (always `"program"`, distinguishing program output from `cprimed`/`cprimec` log lines), `msg`. The other providers emit nothing.

#### Gaps

None.
