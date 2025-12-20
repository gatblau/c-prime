# Component spec — cprime-provider

> Spec set: cprime
> Status: Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> Last updated: 2025-12-20

### SPEC: cprime-provider
**File:** `crates/cprime-provider/src/lib.rs`, `crates/cprime-provider/src/rt.rs`, `crates/cprime-provider/src/sdk.rs`, `crates/cprime-provider/src/registry.rs` | **Package:** `cprime-provider` | **Phase:** 2 (runtime ABI), 3 (SDK) | **Dependencies:** none in the workspace (A-13); third-party: `libc`, `serde_json`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprime-provider` is the one crate every C′ runtime links. It has two halves. The **runtime ABI** (`rt` module) defines the heap object layout, the allocator with memory-limit accounting, the reference-count operations, the byte and vector intrinsics, structural equality, the trap entry point and release-binary initialisation; JIT code, AOT code and the IR interpreter call these functions, so all three back ends share one memory model. The **provider SDK** (`sdk` and `registry` modules) defines what a provider crate declares (one capability's operations with C′ signatures and `deterministic` flags), the uniform C calling convention of every operation, typed helpers for reading arguments and building results, panic containment, and validation of the provider registry. Built-in and third-party providers are written on the same SDK (ADR-020, ADR-021).

#### Approach

The runtime ABI lives in the SDK crate because boundary conversion must allocate C′ heap objects: a provider that returns `Bytes` calls the same allocator the compiled program uses, so placing both in one crate removes a second runtime crate and keeps third-party providers dependent on exactly one C′ crate. Every operation has one C signature (`OpFn`) over 64-bit words, so the JIT needs a single Cranelift signature and AOT code calls operations through one exported table pointer (`cprime_rt_op_table`, an `AtomicPtr<OpFn>` exported by the generated runtime), avoiding per-operation exported symbols. Rejected: a procedural macro generating per-operation `extern "C"` shims — it would need a separate proc-macro crate and symbol names that are only known after the provider crate is compiled. Rejected: recursive `cprime_rt_drop` — a long list built by recursion would overflow the native stack on drop; drop and equality use an explicit heap-allocated work stack. Third-party dependencies are limited to `libc` (`write`, `_exit`, `setrlimit`, `sigaction` for async-signal-safe trap exits and the CPU-time limit) and `serde_json` (`describe_json`); no other crate is permitted, so third-party providers pull in only these two.

#### Shared Context

Copied from `phase-2-architecture.md` §2C and §2E.

```rust
/// Types allowed at the provider boundary (SA-22 adds Unit).
#[derive(Clone, PartialEq, Eq)]
pub enum BoundaryType { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<BoundaryType>), Result(Box<BoundaryType>, Box<BoundaryType>) }

/// Uniform C ABI of every provider operation: arguments are 64-bit words (SA-25), result is one word.
pub type OpFn = unsafe extern "C" fn(args: *const u64, rt: *mut RtContext) -> u64;

#[derive(Clone)]
pub struct OpDecl {
    pub name: &'static str,                      // operation name, e.g. "hmac_sha256"
    pub params: Vec<BoundaryType>,
    pub ret: BoundaryType,
    /// (D40, ADR-054) Provider obligation when true: equal results for equal arguments, no observable effect
    /// (no I/O, no state visible to later calls or to other code), and return without waiting on anything
    /// outside the call. cprimed may call such an operation while checking a contract (fuzzing, concrete replay).
    pub deterministic: bool,
    pub func: OpFn,
    /// (D40) Known-answer vectors, taken from a standard where one exists. Empty is allowed; must be empty
    /// when a parameter or the result is Vec or Result. Checked by sdk::check_vectors at runtime build.
    pub vectors: &'static [KnownAnswer],
}
/// (D40) Arguments in parameter order and the expected result.
#[derive(Clone, Copy)]
pub struct KnownAnswer { pub args: &'static [Lit], pub ret: Lit }
/// (D40) A literal boundary value; Bytes also encodes FixedBytes[N] (its length must equal N).
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum Lit { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(&'static [u8]) }
#[derive(Clone)]
pub struct ProviderDecl { pub path: &'static str, pub crate_name: &'static str, pub crate_version: &'static str, pub ops: Vec<OpDecl>, pub thread_safe: bool, pub transport: Option<TransportDecl> }
/// (D30, ADR-051, SA-66) Transport interface; one word per record field, in declaration order.
#[repr(u32)] #[derive(Clone, Copy, PartialEq, Eq)]
pub enum RecvStatus { Request = 0, Shutdown = 1, Error = 2, Pending = 3 /* reserved; v1 treats it as Error */ }
pub type TransportStartFn = unsafe extern "C" fn(config: *const u8, config_len: usize, err: *mut u8, err_cap: usize) -> u32;
pub type TransportRecvFn  = unsafe extern "C" fn(worker: u32, fields: *mut u64, token: *mut u64, rt: *mut RtContext) -> RecvStatus;
pub type TransportReplyFn = unsafe extern "C" fn(token: u64, fields: *const u64, rt: *mut RtContext);
pub type TransportFailFn  = unsafe extern "C" fn(token: u64, trap: u32);
pub type TransportStopFn  = unsafe extern "C" fn();
#[derive(Clone)]
pub struct TransportDecl {
    pub version: u32,
    pub request: Vec<(&'static str, BoundaryType)>,
    pub response: Vec<(&'static str, BoundaryType)>,
    pub start: TransportStartFn, pub recv: TransportRecvFn, pub reply: TransportReplyFn, pub fail: TransportFailFn, pub stop: TransportStopFn,
}

/// Every provider a runtime links. Order: built-ins (Crypto, Clock, Random, Log), then cprime.toml order.
#[derive(Clone)]
pub struct ProviderRegistry { pub providers: Vec<ProviderDecl> }

#[repr(C)]
pub struct RtContext {      // field order is ABI: JIT and AOT code address fields by offset
    pub trap: u32,          // offset 0:  RtTrap code; 0 = none
    pub mode: u32,          // offset 4:  0 = abort on trap (release), 1 = record and return (interpreter, JIT), 2 = worker (park on trap; par_map), 3 = server task (unwind to the task's recovery point on trap; ADR-051)
    pub fuel: u64,          // offset 8:  remaining function entries; u64::MAX = unlimited
    pub depth: u64,         // offset 16: current C′ call depth
    pub max_depth: u64,     // offset 24: call depth at which a call traps StackLimit
    pub mem_limit: u64,     // offset 32: bytes
    pub mem_live: u64,      // offset 40: bytes currently allocated through this context
    pub tracker: *mut AllocTracker,   // offset 48: non-null in leak-harness and fuzz runs
    pub site: u64,          // offset 56: site id of the Perform in progress, stored by AOT code (D27); 0 = none
    pub chain: *mut u32,    // offset 64: call-chain buffer of max_depth + 1 function indices (D28); null = no call chain
}

/// One entry of the trap site table or the function table generated into a release binary (D27).
pub struct SiteEntry {
    pub spec: &'static str,     // qualified spec name, e.g. "Auth::generate_token"
    pub file: &'static str,     // project-relative .opq path, e.g. "auth/generate_token.opq"
    pub line: u32,              // 1-based
    pub column: u32,            // 1-based, Unicode scalar values
    pub op: &'static str,       // "<Cap>::<op>" for Perform sites; empty otherwise
}

#[repr(u32)] #[derive(Clone, Copy, PartialEq, Eq)]
pub enum RtTrap { None = 0, Overflow = 1, DivisionByZero = 2, IndexOutOfBounds = 3, Unreachable = 4,
                  FuelExhausted = 5, MemoryLimit = 6, ProviderPanic = 7, StackLimit = 8 }
```

Heap object layout (all heap values; little-endian; 8-byte aligned):

| Offset | Size | Field |
|---|---|---|
| 0 | 8 | `rc: u64` — non-atomic count, 1 on allocation |
| 8 | 4 | `len: u32` — byte count (`Bytes`, `FixedBytes`), element count (`Vec`), field count (struct, variant) |
| 12 | 2 | `tag: u16` — variant tag for sums; 0 otherwise |
| 14 | 1 | `kind: u8` — 0 = byte payload, 1 = `Vec`, 2 = struct, 3 = variant |
| 15 | 1 | reserved, 0 |
| 16 | 8 | `ptr_mask: u64` — bit *i* set when payload word *i* is a heap pointer (struct and variant fields); for `Vec`, bit 0 set means every element is a pointer; 0 for byte payloads |
| 24 | … | payload: bytes, or 8-byte words |

Release exit codes (`ExitCode`, SA-16):

| Code | Meaning |
|---|---|
| 0 | Entry returned; result written to stdout |
| 2 | Standard input is not one JSON object matching the entry parameters |
| 3 | Trap (`Overflow`, `DivisionByZero`, `IndexOutOfBounds`, `Unreachable`); stderr line `cprime: trap <Kind>`, followed by ` at <file>:<line>:<column> in <spec>` when the trap has a site (step 8) |
| 4 | CPU-time limit exceeded, or the limit could not be set at start (`cprime: cannot set cpu time limit`) |
| 5 | Memory limit exceeded |
| 6 | Provider panic |
| 7 | Call-depth limit exceeded (`StackLimit`) |
| 8 | Server only (ADR-051): the shutdown grace period expired with a request still running; stderr `cprime: shutdown grace period expired with <n> request(s) running` |
| 9 | Server only (ADR-051): the transport failed to start; stderr `cprime: transport <path> failed to start: <reason>` (reason truncated to 200 characters) |

Value representation (SA-25): every value is one 64-bit word. `Unit` = 0; `Bool` = 0 or 1; `U8` zero-extended; `U64` as is; `I64` two's-complement bits; heap values (`Bytes`, `FixedBytes[N]`, `Vec[T]`, structs, sum values) = address of the object header.

Prelude `Result[T, E] = Ok { value: T } | Err { error: E }`: tag `Ok` = 0, `Err` = 1; each variant has one field.

Runtime protocols (§2E, binding):

- **Trap modes.** `rt.mode == 0`: `cprime_rt_trap` writes the stderr line and exits with the `ExitCode` for the kind. `rt.mode == 1`: `cprime_rt_trap` records the code in `rt.trap` and returns; the caller returns 0 immediately. AOT trap edges call `cprime_rt_trap_at(rt, code, site)`, which in mode 1 behaves as `cprime_rt_trap`.
- **Fuel.** Function entries decrement `rt.fuel` unless it is `u64::MAX` (emitted by `cprime-codegen`, not by this crate).
- **Call depth.** Function entries increment `rt.depth` and returns decrement it; an entry that would make `depth > max_depth` traps `StackLimit` (emitted by `cprime-codegen`). Code runs on a thread whose stack is at least `max_depth × 1 KiB + 1 MiB`; the release `main` generated by `cprime-cli` spawns such a thread.
- **Ownership.** Arguments to runtime functions and to provider operations are borrowed; their results are owned (count 1, or a scalar).
- **Reuse.** `cprime_rt_drop_reuse(rt, x)`: when `rc == 1`, drop `x`'s heap children and return `x`'s address as the token; otherwise decrement `rc` and return 0. The count-is-1 test runs here, at run time.
- **Leak accounting.** `AllocTracker { live_objects: u64, live_bytes: u64, objects: HashSet<usize> }`. When `rt.tracker` is non-null every allocation and free is recorded. `AllocTracker::free_all(&mut self, rt)` releases every tracked object.

Configuration keys read through `cprime_rt_release_init` (values passed by the generated release `main`, never read from a file here):

| Key | Type | Default | Use |
|---|---|---|---|
| `limits.cpu_time_ms` | u64, 1 … 86,400,000 | 10000 | `RLIMIT_CPU`, rounded up to whole seconds |
| `limits.memory_bytes` | u64, 1,048,576 … 2^40 | 1073741824 | `rt.mem_limit` in release binaries |
| `limits.max_call_depth` | u64, 1,000 … 10,000,000 | 100000 | `rt.max_depth` in release binaries |
| `limits.workers` | u32, 0 … 256 | 0 | `par_map` worker count in release binaries; 0 = available parallelism, capped at 256 (D20) |
| `limits.request_fuel` | u64, 1 … 2^63 | 100000000 | Function entries per server task (ADR-051) |
| `limits.shutdown_grace_ms` | u64, 0 … 600,000 | 10000 | Server drain time after `SIGTERM`/`SIGINT` |

`release.call_chain` (bool, default `false`) is not read here. It decides whether the generated release main allocates a chain buffer and stores it in `rt.chain` (D28); this crate reports a chain exactly when `rt.chain` is non-null.

**Release trap locations and call chain (RFC 03, D27, D28, ADR-048).**
- **D27:** AOT code calls `cprime_rt_trap_at(rt, code, site)` on every trap edge and stores the site id of each `Perform` in `rt.site` before the call. The generated release main registers the trap site table and the function table once, through `set_report_tables`, before the entry runs. Site id *s* names `sites[s − 1]`; function index *k* names `functions[k − 1]`; 0 names nothing.
- **D28:** in builds with `release.call_chain`, AOT function prologues write their function index to `rt.chain[rt.depth]`. This crate prints the chain on mode-0 exits 3 to 7, and gives each `par_map` worker context its own buffer, prefilled with the caller's entries `1..=depth`.
- No release output contains a program value: the tables hold names and source locations only (SA-61).

#### Public Interface

All items are re-exported from the crate root.

**Runtime ABI — `cprime_provider::rt`.** Every function below is `#[no_mangle] pub unsafe extern "C"`. All pointer arguments named `ptr`, `a`, `b`, `v`, `x` (when heap-typed) are object header addresses passed as `u64`; 0 is never a valid object.

| Function | Signature | Semantics |
|---|---|---|
| `cprime_rt_alloc` | `(rt: *mut RtContext, payload_bytes: u64, len: u64, tag: u64, kind: u64, ptr_mask: u64) -> u64` | Allocates `24 + round_up_8(payload_bytes)` bytes, 8-byte aligned; writes header (`rc = 1`, `len`, `tag`, `kind`, reserved 0, `ptr_mask`); zero-fills the payload; returns the address. Traps `MemoryLimit` and returns 0 when `len > u32::MAX`, `tag > u16::MAX`, `kind > 3`, `payload_bytes > 2^40`, or `mem_live + size > mem_limit`. Returns 0 without effect when `rt.trap != 0` on entry. |
| `cprime_rt_dup` | `(ptr: u64)` | `rc += 1`. No trap. `ptr == 0` is ignored. |
| `cprime_rt_drop` | `(rt: *mut RtContext, ptr: u64)` | Decrements `rc`; when it reaches 0, frees the object and decrements every heap child, iteratively (explicit work stack). `ptr == 0` and child words equal to 0 are skipped. |
| `cprime_rt_drop_reuse` | `(rt: *mut RtContext, ptr: u64) -> u64` | Reuse protocol: `rc == 1` → drop every heap child, leave the cell allocated and accounted, return `ptr` (the token). `rc > 1` → `rc -= 1`, return 0. |
| `cprime_rt_eq` | `(a: u64, b: u64) -> u64` | Structural equality of two heap values of the same C′ type; returns 1 or 0. Iterative. |
| `cprime_rt_concat` | `(rt: *mut RtContext, a: u64, b: u64) -> u64` | New `Bytes` = `a` bytes followed by `b` bytes (`a`, `b` are kind 0). Traps `MemoryLimit` when the result length exceeds `u32::MAX` or the limit. |
| `cprime_rt_u64_be` | `(rt: *mut RtContext, x: u64) -> u64` | New `Bytes` of length 8, big-endian encoding of `x`. |
| `cprime_rt_push` | `(rt: *mut RtContext, v: u64, x: u64) -> u64` | New `Vec` with `v`'s elements followed by `x`; copies `v`'s `ptr_mask`; when bit 0 is set, every copied element and `x` receive `cprime_rt_dup`. Traps `MemoryLimit` on length or limit overflow. |
| `cprime_rt_slice` | `(rt: *mut RtContext, b: u64, start: u64, end: u64) -> u64` | New `Bytes` = `b[start..end]`. Traps `IndexOutOfBounds` unless `start <= end <= len(b)`. |
| `cprime_rt_to_bytes` | `(rt: *mut RtContext, b: u64) -> u64` | New `Bytes` with the same contents as the `FixedBytes[N]` value `b`. |
| `cprime_rt_trap` | `(rt: *mut RtContext, code: u32)` | Trap entry, per trap mode. In mode 0 the location is taken from `rt.site` for `ProviderPanic` and is absent for every other code (step 8). |
| `cprime_rt_trap_at` | `(rt: *mut RtContext, code: u32, site: u32)` | Trap entry with a site id (D27), called by AOT trap edges. Mode 1 and mode 2: as `cprime_rt_trap`, with `site` recorded by mode 2 (step 15 g). Mode 0: the stderr lines of step 8 for `site`, then `_exit`. |
| `cprime_rt_par_map` | `(rt: *mut RtContext, f: u64, owns_arg: u64, ret_is_ptr: u64, xs: u64) -> u64` | Parallel map (D19, ADR-044). Sequential when `rt.mode ≠ 0`, workers = 1 or `len(xs) < 2`; else parallel over worker contexts. Returns a new `Vec` with `ptr_mask` bit 0 = `ret_is_ptr`. |
| `cprime_rt_serve` | `(rt: *mut RtContext, handler: u64, req_fields: u32, resp_fields: u32, resp_layout: *const u8, transport: u32, config: *const u8, config_len: usize) -> u32` | Server dispatch loop (D33, ADR-051): starts the transport, runs one task per request on the worker threads, drains them on `SIGTERM`/`SIGINT`, and returns the process exit code (0, 8 or 9). Called only by a generated server main. |
| `cprime_rt_release_init` | `(rt: *mut RtContext, cpu_time_ms: u64, memory_bytes: u64, max_call_depth: u64, workers: u64) -> u32` | Initialises a release context (memory and call-depth limits), the `par_map` worker count (0 = available parallelism, capped at 256; stored in a process-wide static) and the CPU-time limit; returns 0 on success, 1 when `setrlimit` or `sigaction` fails. |

Rust-side items in `rt` (not `extern "C"`):

```rust
impl RtContext {
    /// Release context: mode 0, fuel u64::MAX, depth 0, max_depth = max_call_depth, tracker null, site 0, chain null.
    /// The generated release main sets `chain` afterwards when the build has release.call_chain (D28).
    pub fn release(memory_bytes: u64, max_call_depth: u64) -> RtContext;
    /// Interpreter / JIT / fuzz context: mode 1, depth 0, site 0, chain null.
    pub fn sandbox(fuel: u64, max_depth: u64, mem_limit: u64, tracker: *mut AllocTracker) -> RtContext;
    /// Current trap, if any.
    pub fn trap_code(&self) -> RtTrap;
    /// Clears a recorded trap; used by the fuzz harness between iterations.
    pub fn clear_trap(&mut self);
}

/// (D27) Registers the trap site table and the function table of a release binary. Called once by the
/// generated release main before cprime_rt_release_init. Returns true when the tables were stored, and
/// false (storing nothing) when tables were already set.
pub fn set_report_tables(sites: &'static [SiteEntry], functions: &'static [SiteEntry]) -> bool;

pub struct AllocTracker { pub live_objects: u64, pub live_bytes: u64, pub objects: HashSet<usize> }
impl AllocTracker {
    pub fn new() -> AllocTracker;
    /// Frees every tracked cell without following child pointers (children are tracked too);
    /// decrements rt.mem_live by the freed sizes; leaves live_objects == 0 and live_bytes == 0.
    pub unsafe fn free_all(&mut self, rt: &mut RtContext);
}

/// Size in bytes of an allocated cell given its header (24 + payload rounded up to 8).
pub unsafe fn cell_size(ptr: u64) -> u64;
```

**Provider SDK — `cprime_provider::sdk`.**

```rust
/// Implemented by one zero-sized type per provider crate.
pub trait Provider { fn decl() -> ProviderDecl; }

/// Convention: every provider crate exposes at its root
///     pub fn cprime_provider() -> cprime_provider::ProviderDecl
/// which returns `<TheProvider as Provider>::decl()`. The generated project runtime calls it.

// Argument readers. `args` points to `params.len()` words; `i` is the parameter position.
pub unsafe fn arg_word(args: *const u64, i: usize) -> u64;
pub unsafe fn arg_bool(args: *const u64, i: usize) -> bool;        // word != 0
pub unsafe fn arg_u8(args: *const u64, i: usize) -> u8;            // low 8 bits
pub unsafe fn arg_u64(args: *const u64, i: usize) -> u64;
pub unsafe fn arg_i64(args: *const u64, i: usize) -> i64;          // bit reinterpretation
pub unsafe fn arg_bytes<'a>(args: *const u64, i: usize) -> &'a [u8];   // Bytes or FixedBytes<N>; valid only during the call
pub unsafe fn arg_vec_words<'a>(args: *const u64, i: usize) -> &'a [u64]; // Vec element words; valid only during the call

// Result builders. Every builder returns an owned word.
pub fn ret_unit() -> u64;                                          // 0
pub fn ret_bool(b: bool) -> u64;
pub fn ret_u8(x: u8) -> u64;
pub fn ret_u64(x: u64) -> u64;
pub fn ret_i64(x: i64) -> u64;
pub unsafe fn ret_bytes(rt: *mut RtContext, data: &[u8]) -> u64;  // kind 0; used for Bytes and FixedBytes<N>
/// New Vec taking ownership of `words` (owned words); `elems_are_ptrs` sets ptr_mask bit 0.
pub unsafe fn ret_vec(rt: *mut RtContext, words: &[u64], elems_are_ptrs: bool) -> u64;
/// Prelude Result: Ok { value } (tag 0) / Err { error } (tag 1); takes ownership of the payload word.
pub unsafe fn ret_result_ok(rt: *mut RtContext, value: u64, value_is_ptr: bool) -> u64;
pub unsafe fn ret_result_err(rt: *mut RtContext, error: u64, error_is_ptr: bool) -> u64;

/// Runs an operation body; a Rust panic becomes trap ProviderPanic and the word 0.
pub fn guard<F: FnOnce() -> u64 + std::panic::UnwindSafe>(rt: *mut RtContext, f: F) -> u64;

/// (D40) Runs every known-answer vector of every operation; called by `cprimed --check-providers`.
pub fn check_vectors(registry: &ProviderRegistry) -> Vec<VectorFailure>;
pub struct VectorFailure { pub provider: &'static str, pub op: &'static str, pub index: u32, pub detail: String }
```

**Registry — `cprime_provider::registry`.**

```rust
impl ProviderRegistry {
    pub fn flat_ops(&self) -> Vec<(&ProviderDecl, &OpDecl)>;
    pub fn describe_json(&self) -> serde_json::Value;
}

pub fn validate_registry(reg: &ProviderRegistry) -> Result<(), RegistryError>;

pub enum RegistryError {
    InvalidProviderPath { path: String },
    ReservedProviderPath { path: String, crate_name: String },
    DuplicateProvider { path: String },
    EmptyProvider { path: String },
    InvalidOpName { path: String, op: String },
    DuplicateOp { path: String, op: String },
    InvalidBoundaryType { path: String, op: String, detail: String },
}
impl std::fmt::Display for RegistryError { /* messages in the Error Table */ }
```

`describe_json` output (exact keys and order; `index` is the `flat_ops` position; type strings use `.cprime` syntax). `"transport"` is `null`, or, for a transport provider (D30), `{"version": 1, "request": [["method", "Bytes"], …], "response": [["status", "U64"], …]}`:

```json
{
  "format": 1,
  "providers": [
    {
      "path": "host::Crypto",
      "crate": "cprime-host-crypto",
      "version": "0.1.0",
      "thread_safe": true,
      "ops": [
        { "name": "sha256", "index": 0, "params": ["Bytes"], "ret": "FixedBytes[32]", "deterministic": true, "vectors": 2 }
      ],
      "transport": null
    }
  ]
}
```

Type string grammar: `Unit`, `Bool`, `U8`, `U64`, `I64`, `Bytes`, `FixedBytes[N]`, `Vec[T]`, `Result[T, E]` (one space after the comma).

##### Example

A third-party provider crate `cprime-zstd-provider` exposing `acme::Zstd` with `compress(Bytes) Bytes`. The `zstd` crate is that provider's own dependency, not the toolchain's.

```rust
// cprime-zstd-provider/src/lib.rs
use cprime_provider::{sdk::*, BoundaryType as B, OpDecl, ProviderDecl, RtContext};

pub struct Zstd;

unsafe extern "C" fn compress(args: *const u64, rt: *mut RtContext) -> u64 {
    guard(rt, || {
        let input = arg_bytes(args, 0);                 // borrowed; not retained after return
        let out = zstd::bulk::compress(input, 3).expect("zstd level 3 is valid");
        ret_bytes(rt, &out)                             // owned C′ Bytes, count 1
    })
}

impl cprime_provider::sdk::Provider for Zstd {
    fn decl() -> ProviderDecl {
        ProviderDecl {
            path: "acme::Zstd",
            crate_name: env!("CARGO_PKG_NAME"),
            crate_version: env!("CARGO_PKG_VERSION"),
            ops: vec![OpDecl { name: "compress", params: vec![B::Bytes], ret: B::Bytes, deterministic: true, func: compress,
                               vectors: &[] }],   // zstd output is not fixed by a standard, so no vectors (D40)
            thread_safe: false,
            transport: None,
        }
    }
}

pub fn cprime_provider() -> ProviderDecl { <Zstd as cprime_provider::sdk::Provider>::decl() }
```

Matching `.cprime` declaration and binding: `capability Compress { deterministic compress(data Bytes) Bytes }` and `bind Compress = acme::Zstd`. `compress` may be `deterministic` because it has no observable effect and returns without waiting on anything outside the call (ADR-054); with no vectors, `cprimec runtime build` logs a warning and the inventory shows `"vectors": 0`.

Runtime call sequence for `(++ a b)` where `a = 0x0102`, `b = 0x03`, mode 1, `mem_limit = 1024`, `mem_live = 0`:

```text
cprime_rt_alloc(rt, 2, 2, 0, 0, 0)  -> A   ; mem_live = 32 (24 + 8)
cprime_rt_alloc(rt, 1, 1, 0, 0, 0)  -> B   ; mem_live = 64
cprime_rt_concat(rt, A, B)          -> C   ; C.len = 3, payload 01 02 03; mem_live = 96
cprime_rt_drop(rt, A)                      ; rc 1 -> 0, freed; mem_live = 64
```

**Sample transport (not built in; ADR-051).** `cprime-http-transport` declares the provider `acme::HttpTransport` with `thread_safe: true` and `transport: Some(TransportDecl { version: 1, request: [("method", Bytes), ("path", Bytes), ("query", Bytes), ("headers", Vec(Bytes)), ("body", Bytes)], response: [("status", U64), ("headers", Vec(Bytes)), ("body", Bytes)], … })`. Headers are `name: value` lines; `status` outside 100 … 599 is replied as 500. It serves HTTP/1.1 from a configured address, queues at most `4 × workers` parsed requests, refuses further connections with 503, and logs nothing about request contents. It is the worked example for the transport interface and the sample for RFC 06 criteria 5–9.

#### Internal Logic

1. **Allocation (`cprime_rt_alloc`).** (a) If `rt.trap != 0`, return 0. (b) Validate `len <= u32::MAX`, `tag <= u16::MAX`, `kind <= 3`, `payload_bytes <= 2^40`; on failure call `cprime_rt_trap(rt, MemoryLimit)` and return 0. (c) `size = 24 + ((payload_bytes + 7) & !7)`. (d) If `rt.mem_live + size > rt.mem_limit`, trap `MemoryLimit`, return 0. (e) Allocate with `std::alloc::alloc_zeroed(Layout::from_size_align(size, 8))`; a null return traps `MemoryLimit` and returns 0. (f) Write the header; `rt.mem_live += size`. (g) When `rt.tracker` is non-null: insert the address into `objects`, `live_objects += 1`, `live_bytes += size`. No logging.
2. **Free (internal `free_cell`).** Compute `cell_size(ptr)` from `kind` and `len` (kind 0: `24 + round_up_8(len)`; kinds 1–3: `24 + 8 * len`), `std::alloc::dealloc`, `rt.mem_live -= size`, and when tracked remove from `objects`, `live_objects -= 1`, `live_bytes -= size`. A `free_cell` on an address absent from a non-null tracker is a runtime bug and triggers `debug_assert!` in debug builds; release builds skip the tracker update.
3. **Dup (`cprime_rt_dup`).** If `ptr != 0`, increment `rc` with wrapping-free addition (`rc` cannot reach `u64::MAX` because each reference needs a word of memory).
4. **Drop (`cprime_rt_drop`).** Push `ptr` on a work stack (`Vec<u64>`, initial capacity 16). Loop: pop `p`; skip 0; `rc -= 1`; if `rc > 0` continue; otherwise, for kinds 1–3 push every payload word *i* that is a pointer (struct/variant: bit *i* of `ptr_mask`; `Vec`: all words when bit 0 is set) and is non-zero; then `free_cell(p)`. The stack lives on the Rust heap and is not charged to `mem_live`. No trap is possible.
5. **Drop-reuse (`cprime_rt_drop_reuse`).** If `rc > 1`: `rc -= 1`, return 0. If `rc == 1`: run step 4's child loop on each heap child of `ptr` (not on `ptr` itself), set every payload word to 0, return `ptr`. The cell stays allocated and counted in `mem_live`; the consuming `Make*` overwrites the header and payload. An unconsumed token on a trap path is released by `AllocTracker::free_all`.
6. **Equality (`cprime_rt_eq`).** Push `(a, b)` on a pair stack. Loop: pop; if addresses are equal continue; compare `kind`, `len`, `tag`; for kind 0 compare payload bytes; for kinds 1–3 compare words: pointer words (per `ptr_mask`) push a pair, scalar words compare by value. Any mismatch returns 0; an empty stack returns 1. Callers guarantee both values have the same C′ type (type checking).
7. **Byte and vector intrinsics.** `cprime_rt_concat`: lengths `la + lb` computed in `u64`; allocate kind 0 with `len = la + lb`; copy. `cprime_rt_u64_be`: allocate 8 bytes, write `x.to_be_bytes()`. `cprime_rt_push`: allocate kind 1 with `len = len(v) + 1`, `ptr_mask = v.ptr_mask`; copy words; append `x`; when bit 0 is set, `cprime_rt_dup` every copied element and `x`. `cprime_rt_slice`: check `start <= end && end <= len(b)` else trap `IndexOutOfBounds` and return 0; allocate `end - start` bytes; copy. `cprime_rt_to_bytes`: allocate `len(b)` bytes, copy. Each returns 0 without effect when `rt.trap != 0` on entry. Arguments are borrowed: none of these functions changes an argument's `rc`.
8. **Trap (`cprime_rt_trap`, `cprime_rt_trap_at`).** Mode 1: if `rt.trap == 0`, set `rt.trap = code` (the first trap wins); return. `cprime_rt_trap_at` in mode 1 does the same and ignores `site`. Mode 2: step 15 (g).

   Mode 0: write the report lines below to file descriptor 2, then `libc::_exit(code_to_exit)`.
   - **Site.** The site *s* is the `site` argument of `cprime_rt_trap_at`. For `ProviderPanic`, raised through `cprime_rt_trap` by `guard`, it is `rt.site`. For every other call of `cprime_rt_trap`, it is 0. The entry is `sites[s − 1]` of the registered tables. The site is treated as absent when *s* is 0, *s* exceeds `sites.len()`, or no tables are registered.
   - **First line:**

     | `code` | first line, site absent | first line, site present | exit |
     |---|---|---|---|
     | 1 `Overflow` | `cprime: trap Overflow` | `cprime: trap Overflow at <file>:<line>:<column> in <spec>` | 3 |
     | 2 `DivisionByZero` | `cprime: trap DivisionByZero` | `cprime: trap DivisionByZero at <file>:<line>:<column> in <spec>` | 3 |
     | 3 `IndexOutOfBounds` | `cprime: trap IndexOutOfBounds` | `cprime: trap IndexOutOfBounds at <file>:<line>:<column> in <spec>` | 3 |
     | 4 `Unreachable` | `cprime: trap Unreachable` | `cprime: trap Unreachable at <file>:<line>:<column> in <spec>` | 3 |
     | 5 `FuelExhausted` | `cprime: cpu time limit exceeded` | (never has a site) | 4 |
     | 6 `MemoryLimit` | `cprime: memory limit exceeded` | (never has a site) | 5 |
     | 7 `ProviderPanic` | `cprime: provider panic` | `cprime: provider panic in <op> at <file>:<line>:<column> in <spec>` | 6 |
     | 8 `StackLimit` | `cprime: call depth limit exceeded` | `cprime: call depth limit exceeded in <spec> (<file>:<line>:<column>)` | 7 |
     | any other | `cprime: trap Unreachable` | `cprime: trap Unreachable` | 3 |

     `<file>`, `<line>`, `<column>`, `<spec>` and `<op>` are the fields of the `SiteEntry`. Every existing first line is the prefix of its located form, so exit codes and leading text are unchanged (ADR-048), with the prefix `cprime:` (ADR-049). Inside `par_map`, step 15 (e) appends ` (par_map element <i>)` to the first line.
   - **Call chain.** When `rt.chain` is non-null and `rt.depth ≥ 1`, with *d* = `rt.depth`, these lines follow the first line:
     - `cprime: call chain (outermost first, <d> frames):`
     - one line per frame *k* = 1 … *d*, read from `chain[k]`: `cprime:   <spec> (<file>:<line>:<column>)` from `functions[chain[k] − 1]`, or `cprime:   <unknown function>` when the index is 0 or exceeds `functions.len()`;
     - when *d* > 32: frames 1 … 8, then `cprime:   … <d − 32> frames omitted`, then frames *d* − 23 … *d*.
   - **Output mechanics.** Each line is composed in one 512-byte buffer on the stack, with no heap allocation and no formatting machinery that allocates. Integers are written by a fixed digit loop. A line longer than 511 bytes is cut at 510 bytes and ended with `\n`. Each line goes out with one `libc::write`. Only names and source locations are written, never a program value.

   `_exit` is used instead of the Rust standard library exit function, so no destructors or buffered stdout of a partial result run. The same report routine serves the `SIGXCPU` action (step 9) and the `par_map` main thread (step 15 e); it reads `rt.depth` and `rt.chain` with volatile reads.
8a. **Report tables (`set_report_tables`).** The two slices are stored in a process-wide `std::sync::OnceLock`. The first call stores them and returns `true`; a later call stores nothing and returns `false`. Reading the tables during a report is one atomic load, which is async-signal-safe on the two supported targets.
9. **Release initialisation (`cprime_rt_release_init`).** (a) Overwrite `*rt` with `RtContext::release(memory_bytes, max_call_depth)`, keeping a non-null `rt.chain` the generated main set before the call. (a2) Store `rt` in the process-wide `AtomicPtr<RtContext>` `RELEASE_RT` (D28). (b) Install a `SIGXCPU` action with `sigaction`. The action writes `cprime: cpu time limit exceeded\n` with `libc::write`. When `RELEASE_RT` holds a context whose `chain` is non-null and whose `depth ≥ 1`, it then writes that context's call chain by the rules of step 8, using volatile reads and the stack buffer. Finally it calls `libc::_exit(4)`. Every call in the action is async-signal-safe. During a `par_map`, the chain shown is the main thread's, up to the `par_map` call. (c) Call `setrlimit(RLIMIT_CPU)` with soft limit `ceil(cpu_time_ms / 1000)` seconds and hard limit soft + 1 (the kernel sends `SIGKILL` at the hard limit, one second after the caught signal). (c2) Store the worker count in the process-wide static: `workers` when 1 … 256, else (`0`) `min(std::thread::available_parallelism(), 256)`, or 1 when that call fails. (d) Return 0, or 1 if (b) or (c) failed. No stderr output on success.
10. **Panic containment (`guard`).** Calls `std::panic::catch_unwind(f)`. `Ok(w)` returns `w`. `Err(_)` calls `cprime_rt_trap(rt, ProviderPanic)` and returns 0 (mode 0 never returns from the trap). The default panic hook still prints the panic message to stderr. The generated project runtime packages use `panic = "unwind"` (Cargo's default); a provider compiled with `panic = "abort"` aborts the whole program instead, which is recorded in `cprime-cli`'s runtime build.
11. **Argument and result helpers.** `arg_bytes` reads `len` at offset 8 and returns the slice at offset 24; `arg_vec_words` returns `len` words at offset 24. `ret_bytes` calls `cprime_rt_alloc(rt, data.len(), data.len(), 0, 0, 0)` and copies; when allocation returns 0 it returns 0 (a trap is already recorded). `ret_vec` allocates kind 1 with `ptr_mask = elems_are_ptrs as u64` and writes the words without changing their counts. `ret_result_ok`/`ret_result_err` allocate kind 3, `len = 1`, `tag` 0/1, `ptr_mask = is_ptr as u64`, payload word = the given word. Boundary values are copies: a provider must not store a pointer obtained from `arg_*` after it returns, and must not return a borrowed argument word as its result without `cprime_rt_dup` (the SDK documentation states both rules on every helper).
12. **`flat_ops`.** Iterates `providers` in order and each provider's `ops` in declaration order; the position in the returned vector is the global operation index used by `cprime_rt_op_table` and `ProviderOpInfo.flat_index`. `cprime_rt_op_table` itself is not defined in this crate: the generated runtime package (`components/cprime-cli.md`) exports `#[no_mangle] pub static cprime_rt_op_table: AtomicPtr<OpFn>` and stores a heap array of every `OpDecl::func` in `flat_ops()` order through its generated `init_op_table()` before the entry function runs; AOT code loads the pointer and indexes it.
13. **`describe_json`.** Builds the JSON shown in *Public Interface* with `serde_json::json!`, preserving registry order; `index` comes from `flat_ops`; `vectors` is `OpDecl.vectors.len()` (D40).
14. **`validate_registry`.** Checks, in this order, returning the first error: provider path matches `^[A-Za-z][A-Za-z0-9_]*(::[A-Za-z][A-Za-z0-9_]*)*$` (`InvalidProviderPath`); a path whose first segment is `host` belongs to a crate whose name starts with `cprime-host-` (`ReservedProviderPath`); paths are unique (`DuplicateProvider`); each provider has ≥ 1 operation (`EmptyProvider`); each operation name matches `^[a-z][a-z0-9_]*$` (`InvalidOpName`) and is unique within its provider (`DuplicateOp`); every parameter and result type is well formed (`InvalidBoundaryType`): `Unit` is legal only as a result type or as a `Result` argument; a `Vec` element is one of `Bool`, `U8`, `U64`, `I64`, `Bytes`, `FixedBytes[N]`; `Result` arguments are any type except `Result`; `FixedBytes(N)` requires `1 <= N <= 65536`. `cprimed` and `cprimec` call it at start-up and stop on `Err`.
15. **Parallel map (`cprime_rt_par_map`, D19, ADR-044).** Elements are pointers when `xs.ptr_mask` bit 0 is set.
   - (a) *Sequential path* (`rt.mode ≠ 0`, worker count 1, or `len(xs) < 2`): for `i` in `0..len`: `a = xs[i]`; if `owns_arg` and the elements are pointers, `cprime_rt_dup(a)`; `w = f(rt, a)`; if `rt.trap ≠ 0` return 0 (mode 1; protocol 7 frees the rest); collect `w`. Build the result `Vec` with `ptr_mask` bit 0 = `ret_is_ptr`.
   - (b) *Parallel path:* `k = min(workers, len)` contiguous chunks whose sizes differ by at most 1. Record the caller's `mem_live` in the pool state. Spawn `k` scoped threads (`std::thread::scope`), each with stack `max_depth × 1 KiB + 1 MiB`. Worker `j` builds `RtContext { trap 0, mode 2, fuel u64::MAX, depth rt.depth, max_depth rt.max_depth, mem_limit rt.mem_limit, mem_live 0, tracker null, site 0, chain null }` and a thread-local worker record `(pool, j, current index)`. When the caller's `rt.chain` is non-null (D28), the worker allocates its own buffer of `max_depth + 1` `u32` entries on the Rust heap (not charged to `mem_live`), copies the caller's entries `1..=rt.depth` into it, and sets its context's `chain` to that buffer. The buffer lives until the pool is torn down. For each `i` in its chunk: set the current index; `a = copy_value(wrt, xs[i])` (pointer elements only); `w = f(wrt, a)`; if parameter 0 is borrowed (`owns_arg = 0`), `cprime_rt_drop(wrt, a)`; store `w`. On finishing, signal done.
   - (c) *Copy (`copy_value`):* work-stack traversal over header `kind`, `len`, `tag`, `ptr_mask`, memoised by source address so shared children are copied once. Each destination child's `rc` equals its number of parents within the copy. Allocates through the destination context. Never reads or writes a source `rc`.
   - (d) *Wait:* the main thread blocks on a condition variable until every worker is done or parked.
   - (e) *Trap:* if a trap was recorded, the main thread reports the record of the lowest recorded index `(index i, code, site, worker j)`. It writes the step 8 lines for `code` and `site`, with ` (par_map element <i>)` appended to the first line. In chain builds it prints worker `j`'s chain: worker `j` is parked, so its context and buffer are still valid. Then it calls `_exit` with the code's exit code. In modes 1 and 2 the sequential path applies and no worker exists.
   - (f) *Success:* copy each result into the caller's context in index order (`copy_value(rt, w)` for pointer results), then `cprime_rt_drop(wrt_j, w)`; every worker context's `mem_live` must then be 0 (`debug_assert`). Join the threads and return the new `Vec`.
   - (g) *Mode 2 in `cprime_rt_trap` and `cprime_rt_trap_at`:* under the pool's mutex, replace the recorded `(index, code, site, worker)` when this worker's current index is lower than the recorded one. `site` is the `site` argument of `cprime_rt_trap_at`; for `ProviderPanic` it is `wrt.site`, otherwise 0. Signal the condition variable, then `loop { std::thread::park() }`.
   - (h) *Mode 2 in `cprime_rt_alloc`/`cprime_rt_drop`:* besides `wrt.mem_live`, add or subtract `size` on the process-wide `AtomicU64` counter. Allocation traps `MemoryLimit` when `counter + caller_mem_live > mem_limit`, after subtracting its own `size` back.
   - (i) *Nesting:* `cprime_rt_par_map` with `rt.mode == 2` takes the sequential path.
   - (j) *Spawn failure:* a worker thread that cannot be spawned → stderr `cprime: cannot start worker thread`, `_exit(3)`, before any element runs.
16. **Server dispatch (`cprime_rt_serve`, D33, ADR-051, SA-67, SA-68).**
   - (a) *Start.* Look up `registry.providers[transport].transport` (absent → stderr `cprime: transport <path> failed to start: not a transport`, return 9). Call `start(config, …)`; non-zero → stderr `cprime: transport <path> failed to start: <reason>` (reason truncated to 200 characters), return 9. Install `SIGTERM` and `SIGINT` handlers that store `true` in a static `AtomicBool` `STOPPING` and call the transport's `stop` (both async-signal-safe).
   - (b) *Workers.* `k = limits.workers` (0 → available parallelism, capped at 256). Spawn `k` threads (stack `max_depth × 1 KiB + 1 MiB`); spawn failure → stderr `cprime: cannot start worker thread`, call `stop`, return 3. The main thread waits on a condition variable.
   - (c) *Task loop (each worker `j`).* Repeat:
     1. Build a task context `trt = RtContext { trap 0, mode 3, fuel: limits.request_fuel, depth 0, max_depth, mem_limit: rt.mem_limit, mem_live 0, tracker: <a fresh AllocTracker>, site 0, chain: <worker buffer or null> }`. `mem_live` updates also add to the process-wide `AtomicU64` live counter, which enforces `mem_limit` across all tasks.
     2. Call `recv(j, fields, &token, trt)`. `Shutdown` → leave the loop. `Error` or `Pending` → stderr `cprime: transport recv failed`, then continue with the next iteration.
     3. Build the request struct in the task heap from `fields` (one cell, `req_fields` words, pointer mask from the request layout).
     4. Call `std::panic::catch_unwind(|| handler(trt, req))`.
        - **Normal return `w`:** read the response fields from `w` per `resp_layout`, call `reply(token, fields, trt)`, then `cprime_rt_drop(trt, w)`. The task heap must now be empty (`debug_assert!(trt.tracker.live_objects == 0)`).
        - **Unwind with a `TaskTrap { code, site }` payload:** call `fail(token, code)`, then `tracker.free_all(trt)`.
        - **Any other panic payload:** a provider panic outside `guard`. Treat it as `ProviderPanic` (code 7): `fail(token, 7)`, then `free_all`.
   - (d) *Mode 3 trap entry.* `cprime_rt_trap` and `cprime_rt_trap_at` are `extern "C-unwind"`. In mode 3 they:
     1. write the step 8 lines for `code` and `site`, with ` (request failed)` appended to the first line (no request data);
     2. call `std::panic::resume_unwind(Box::new(TaskTrap { code, site }))`.

     `FuelExhausted`, `StackLimit`, `MemoryLimit` and `ProviderPanic` raised inside the task take the same path. The trap entry holds no values with destructors. AOT frames own only task-heap cells, which `free_all` reclaims.
   - (e) *Shutdown.* After `STOPPING` is set, the main thread waits up to `limits.shutdown_grace_ms` for every worker to leave its loop.
     - All left → return 0.
     - Otherwise → stderr `cprime: shutdown grace period expired with <n> request(s) running`, return 8. The generated main exits with it, abandoning the running tasks.
   - (f) *No CPU limit.* `cprime_rt_release_init` is called with `cpu_time_ms = 0` by a server main, which skips `RLIMIT_CPU`.


17. **Known-answer check (`sdk::check_vectors`, D40, ADR-054).** For each `(provider, op)` in `flat_ops` order and each vector *k* of `op.vectors` (index from 0):
    - (a) If a parameter or the result of `op` is `Vec` or `Result` → `VectorFailure { detail: "vectors are not supported for this signature" }` for *k*; next vector.
    - (b) If `args.len()` differs from `op.params.len()`, or a `Lit` does not match its boundary type (a `Bytes` literal for `FixedBytes[N]` must have length *N*) → `detail` `vector does not match the signature`.
    - (c) Twice: create `AllocTracker::default()` and `RtContext::sandbox(1_000_000, 50_000, 67_108_864, &mut tracker)`; convert each `Lit` to a word (`Unit` 0; scalars as `ret_*`; `Bytes` with `ret_bytes`); call `(op.func)(args.as_ptr(), &mut rt)`; a non-zero `rt.trap` → `detail` `trap <RtTrap name>`; otherwise read the result word back as the `Lit` kind of `ret` (bytes copied from the cell); then `tracker.free_all(&mut rt)`.
    - (d) First result ≠ `ret` → `detail` `result differs: expected <hex or decimal>, got <hex or decimal>`; second result ≠ first → `detail` `results differ between two calls`.
    - Returns every failure in order; an empty `Vec` means every vector passed. No logging; `cprimed --check-providers` renders the failures.
#### Data Model

N/A — no persistent storage. In-memory structures:
- heap cells (layout in *Shared Context*);
- `RtContext` (one per program run, per fuzz call, per interpreter run, per `par_map` worker, or per server task);
- per server task, one `AllocTracker` recording its task heap, freed in full when the task ends abnormally (step 16);
- `AllocTracker` (one per leak-harness or fuzz run);
- the static `ProviderRegistry`, built once at start-up by the generated runtime;
- in release binaries:
  - the report tables (`OnceLock<(&'static [SiteEntry], &'static [SiteEntry])>`, step 8a);
  - `RELEASE_RT` (`AtomicPtr<RtContext>`, step 9);
  - in call-chain builds, one chain buffer of `max_depth + 1` `u32` entries per context: the main one allocated by the generated main, one per worker allocated by step 15 (b).

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Allocation would exceed `rt.mem_limit`, or `len`/`tag`/`kind`/size out of range, or the global allocator returns null | mode 1: `rt.trap` set, function returns 0; mode 0: exit 5 | `RtTrap::MemoryLimit` (6) | stderr (mode 0): `cprime: memory limit exceeded` |
| `cprime_rt_slice` with `start > end` or `end > len(b)` | mode 1: `rt.trap` set, returns 0; mode 0: exit 3 | `RtTrap::IndexOutOfBounds` (3) | `cprime: trap IndexOutOfBounds` |
| Compiled code reports a checked-arithmetic or index trap through `cprime_rt_trap` | mode 1: recorded; mode 0: exit 3 | `Overflow` (1), `DivisionByZero` (2), `IndexOutOfBounds` (3), `Unreachable` (4) | `cprime: trap <Kind>` |
| AOT code reports a trap through `cprime_rt_trap_at` with a site present in the table | mode 1: recorded; mode 0: exit per code | codes 1–4, 8 | e.g. `cprime: trap Overflow at auth/generate_token.opq:3:13 in Auth::generate_token`; `cprime: call depth limit exceeded in Math::fib (math/fib.opq:2:1)` |
| `cprime_rt_trap_at` with site 0, a site beyond the table, or no tables registered | mode 0: exit per code | codes 1–4, 8 | the line without a location, e.g. `cprime: trap Overflow` |
| `set_report_tables` called a second time | returns `false` | — | none; the first tables stay in force |
| A chain entry is 0 or beyond the function table | mode 0: reported, exit per code | — | that frame's line reads `cprime:   <unknown function>` |
| A report line exceeds 511 bytes | mode 0: reported, exit per code | — | the line is cut at 510 bytes and ended with `\n` |
| Fuel reaches 0 (interpreter/JIT) | mode 1: recorded | `RtTrap::FuelExhausted` (5) | — (mode 0 never has finite fuel; if reached: `cprime: cpu time limit exceeded`, exit 4) |
| Release binary exceeds `RLIMIT_CPU` soft limit | `SIGXCPU` action, exit 4 | — | `cprime: cpu time limit exceeded`, followed by the main context's call chain in call-chain builds |
| Provider operation body panics inside `guard` | mode 1: recorded, returns 0; mode 0: exit 6 | `RtTrap::ProviderPanic` (7) | panic hook message, then `cprime: provider panic in <op> at <file>:<line>:<column> in <spec>` when `rt.site` names a table entry, else `cprime: provider panic` |
| `cprime_rt_release_init` cannot install the signal action or the CPU limit | returns 1 | — | none from this crate; the generated `main` writes `cprime: cannot set cpu time limit` and exits 4 |
| Provider path fails the path pattern | `Err` from `validate_registry` | `InvalidProviderPath` | `invalid provider path '<path>'` |
| `host::…` path declared by a crate not named `cprime-host-*` | `Err` | `ReservedProviderPath` | `provider path '<path>' is reserved for built-in providers (declared by '<crate>')` |
| Two providers declare the same path | `Err` | `DuplicateProvider` | `provider path '<path>' is declared more than once` |
| Provider with no operations | `Err` | `EmptyProvider` | `provider '<path>' declares no operations` |
| Operation name fails the pattern | `Err` | `InvalidOpName` | `provider '<path>': invalid operation name '<op>'` |
| Operation name repeated in one provider | `Err` | `DuplicateOp` | `provider '<path>': operation '<op>' is declared more than once` |
| A known-answer vector fails, has the wrong shape, or is declared on a `Vec`/`Result` signature (D40) | entry in the `Vec<VectorFailure>` | — | `cprimed --check-providers` prints `cprimed: provider <path> operation <op> fails known-answer vector <k>: <detail>`; `cprimec runtime build` exits 1 |
| Ill-formed boundary type | `Err` | `InvalidBoundaryType` | `provider '<path>' operation '<op>': <detail>` (e.g. `Vec element type Result[U64, U64] is not allowed`) |
| Server transport absent or `start` fails | release: stderr `cprime: transport <path> failed to start: <reason>`, exit 9 | — | step 16 (a) |
| Trap, fuel, call-depth or memory limit inside a server task | the request fails through `fail`; stderr line with ` (request failed)`; the server continues | `RtTrap` code | step 16 (c)–(d) |
| Shutdown grace period expires | stderr `cprime: shutdown grace period expired with <n> request(s) running`, exit 8 | — | step 16 (e) |
| `par_map` worker thread cannot be spawned | release: stderr `cprime: cannot start worker thread`, exit 3 | `RtTrap::Unreachable` | raised by the main thread before any element runs |
| Combined live heap of caller and workers exceeds `mem_limit` | trap in the worker; reported per step 15 (e) | `RtTrap::MemoryLimit` / exit 5 | raised inside `cprime_rt_alloc` in mode 2 |

#### Acceptance Criteria (Gherkin)

Feature: cprime-provider

  Scenario: Happy path — allocate, concatenate and drop with exact accounting
    Given a sandbox context with mem_limit 1024 and a new AllocTracker
    And Bytes A = 0x0102 and Bytes B = 0x03 allocated through cprime_rt_alloc
    When cprime_rt_concat(rt, A, B) returns C
    Then C has kind 0, len 3 and payload 01 02 03
    And mem_live is 96 and tracker.live_objects is 3
    When cprime_rt_drop is called on A, B and C
    Then mem_live is 0 and tracker.live_objects is 0

  Scenario: Happy path — drop of a long list does not overflow the native stack
    Given a sum-typed list of 1,000,000 cells, each a variant whose second field points to the next cell
    When cprime_rt_drop is called on the head with a count of 1
    Then every cell is freed and tracker.live_objects is 0
    And the call returns without a stack overflow

  Scenario: Edge case — drop-reuse with a shared value returns no token
    Given a struct value S with rc 2
    When cprime_rt_drop_reuse(rt, S) is called
    Then the result is 0 and S.rc is 1

  Scenario: Edge case — drop-reuse with a unique value returns its own address
    Given a struct value S with rc 1 whose field 0 points to Bytes X with rc 1
    When cprime_rt_drop_reuse(rt, S) is called
    Then the result equals S's address
    And X is freed and S's payload words are 0
    And mem_live still includes S's cell size

  Scenario: Edge case — structural equality across distinct cells
    Given two separately allocated Vec[Bytes] values [0x01, 0x0203] and [0x01, 0x0203]
    When cprime_rt_eq is called on them
    Then the result is 1
    And comparing [0x01, 0x0203] with [0x01, 0x0204] returns 0

  Scenario: Edge case — push dups pointer elements
    Given v = Vec[Bytes] [X] with X.rc 1 and a Bytes value Y with rc 1
    When cprime_rt_push(rt, v, Y) returns w
    Then w has len 2 and ptr_mask bit 0 set
    And X.rc is 2 and Y.rc is 2 and v is unchanged

  Scenario: Error path — memory limit in sandbox mode
    Given a sandbox context with mem_limit 64 and mem_live 48
    When cprime_rt_alloc(rt, 16, 16, 0, 0, 0) is called
    Then the result is 0 and rt.trap is MemoryLimit (6)
    And a following cprime_rt_alloc returns 0 without allocating

  Scenario: Error path — slice out of range in release mode
    Given a release context and Bytes b of length 4
    When cprime_rt_slice(rt, b, 2, 5) is called
    Then the program writes "cprime: trap IndexOutOfBounds" to stderr
    And the program exits with code 3

  Scenario: Error path — provider panic is contained in sandbox mode
    Given an operation whose body calls panic!("boom") inside guard
    When the operation is called with a sandbox context
    Then the call returns 0 and rt.trap is ProviderPanic (7)

  Scenario: Error path — CPU-time limit in a release binary
    Given cprime_rt_release_init(rt, 1000, 1073741824, 100000, 0) returned 0
    When the program consumes more than 1 second of CPU time
    Then it writes "cprime: cpu time limit exceeded" to stderr and exits with code 4

  Scenario: Error path — duplicate provider path rejected
    Given a registry with two providers declaring path "acme::Zstd"
    When validate_registry is called
    Then it returns DuplicateProvider with path "acme::Zstd"

  Scenario: Error path — Unit as a parameter type rejected
    Given a provider operation with params [Unit]
    When validate_registry is called
    Then it returns InvalidBoundaryType naming the provider and the operation

  Scenario: Happy path — describe_json lists operations in flat order
    Given a registry with host::Crypto (sha256, hmac_sha256, blake3) followed by acme::Zstd (compress)
    When describe_json is called
    Then acme::Zstd's compress has index 3 and vectors 0
    And the JSON has "format": 1 and providers in registry order

  Scenario: Error path — check_vectors reports a wrong vector and passes a correct one (D40)
    Given a test provider with deterministic operation double(x U64) U64 returning 2x
    And vectors [{args: [U64(2)], ret: U64(4)}, {args: [U64(3)], ret: U64(7)}]
    When check_vectors runs
    Then it returns exactly one VectorFailure with index 1 and a detail starting "result differs"

  Scenario: Error path — a vector on a Vec signature is refused (D40)
    Given an operation sum(xs Vec[U64]) U64 with one vector
    When check_vectors runs
    Then the VectorFailure detail is "vectors are not supported for this signature"

  Scenario: Happy path — results do not depend on the worker count
    Given a release runtime with workers 1, 2 and 4 in turn
    And cprime_rt_par_map over a Vec of 1000 Bytes with a function returning the length of its argument
    Then the three result vectors are equal element by element
    And every worker context's mem_live is 0 after the call

  Scenario: Edge case — shared children are copied once
    Given a Vec of 8 elements that all reference the same Bytes object
    When the parallel path copies one chunk of 4 elements into a worker context
    Then the worker context holds one copy of the Bytes object per element copy, not per reference inside one element

  Scenario: Error path — lowest index trap wins
    Given workers 4 and a function that traps Overflow on elements 3 and 7
    When cprime_rt_par_map runs in a release binary 20 times
    Then every run exits 3 with a stderr first line starting "cprime: trap Overflow"
    And the pool state recorded index 3 on every run

  Scenario: Error path — memory limit across workers
    Given mem_limit 64 MiB and 4 workers that each allocate 20 MiB
    When cprime_rt_par_map runs
    Then the binary exits 5

  Scenario: Error path — overflow with site prints location
    Given a release context and report tables whose site 2 is SiteEntry { spec: "Auth::generate_token", file: "auth/generate_token.opq", line: 3, column: 13, op: "" }
    When cprime_rt_trap_at(rt, 1, 2) is called
    Then stderr is exactly "cprime: trap Overflow at auth/generate_token.opq:3:13 in Auth::generate_token\n"
    And the program exits with code 3

  Scenario: Edge case — site 0 prints the existing line
    Given a release context and the same report tables
    When cprime_rt_trap_at(rt, 1, 0) is called
    Then stderr is exactly "cprime: trap Overflow\n" and the program exits with code 3
    And cprime_rt_trap_at(rt, 8, 99) with only 3 sites registered writes "cprime: call depth limit exceeded\n" and exits 7

  Scenario: Error path — provider panic names the operation
    Given a release context whose rt.site is 3
    And report tables whose site 3 is SiteEntry { spec: "Auth::generate_token", file: "auth/generate_token.opq", line: 5, column: 13, op: "Auth::CryptoCap::hmac_sha256" }
    When an operation body calls panic!("boom") inside guard
    Then stderr holds the panic hook message followed by "cprime: provider panic in Auth::CryptoCap::hmac_sha256 at auth/generate_token.opq:5:13 in Auth::generate_token\n"
    And the program exits with code 6

  Scenario: Edge case — 100-frame chain prints 8 + omitted + 24
    Given a release context with depth 100 and a chain buffer whose entries 1 to 100 all hold function index 1
    And a function table whose entry 1 is SiteEntry { spec: "Math::fib", file: "math/fib.opq", line: 2, column: 1, op: "" }
    When cprime_rt_trap_at(rt, 8, 1) is called with site 1 = SiteEntry { spec: "Math::fib", file: "math/fib.opq", line: 2, column: 1, op: "" }
    Then stderr starts "cprime: call depth limit exceeded in Math::fib (math/fib.opq:2:1)\ncprime: call chain (outermost first, 100 frames):\n"
    And then holds 8 lines "cprime:   Math::fib (math/fib.opq:2:1)", one line "cprime:   … 68 frames omitted" and 24 lines "cprime:   Math::fib (math/fib.opq:2:1)"
    And the program exits with code 7

  Scenario: Error path — CPU limit in a chain build prints the chain
    Given cprime_rt_release_init(rt, 1000, 1073741824, 100000, 0) returned 0 for a context with a non-null chain buffer
    And the program is inside Main::run called from the entry, so depth is 2 with chain entries for Main::main and Main::run
    When the program consumes more than 1 second of CPU time
    Then stderr holds "cprime: cpu time limit exceeded", then "cprime: call chain (outermost first, 2 frames):", then one frame line each for Main::main and Main::run
    And the program exits with code 4

  Scenario: Edge case — output contains no program value
    Given the release binary of the concept example built with release.call_chain true
    And the input {"user_id": 7777777, "now": 18446744073709551615, "secret": "0x" followed by 64 hex digits "ab"}
    When the binary traps Overflow
    Then stderr contains neither "7777777" nor "18446744073709551615" nor "abab"
    And every stderr line starts with "cprime: "

  Scenario: Happy path — server tasks run concurrently
    Given a server runtime with workers 4 on a machine with at least 4 cores and a handler that burns about 1 s of CPU
    When 4 requests arrive at once through a test transport
    Then all 4 replies arrive within 2 s of wall-clock time and each equals the handler's result for its request

  Scenario: Error path — a trapping request fails alone
    Given a server runtime and a handler that overflows on requests whose body is "boom"
    When the requests "boom" and "ok" arrive in that order
    Then the transport's fail is called for "boom" with code 1 and reply is called for "ok"
    And stderr has exactly one line starting "cprime: trap Overflow" ending in "(request failed)" and containing no request data
    And the process keeps running

  Scenario: Edge case — task heaps return to the baseline
    Given a server runtime with workers 2 and a handler that allocates
    When 1,000,000 requests are served, 1% of them trapping
    Then no StackLimit occurs and the process-wide live counter after the run equals its value after the first request

  Scenario: Error path — per-task fuel
    Given limits.request_fuel 1000 and a handler recursing 2000 times on one request
    When that request and a normal one arrive
    Then the first fails with FuelExhausted and the second succeeds

  Scenario: Edge case — shutdown drains, then expires
    Given limits.shutdown_grace_ms 500 and one running request that finishes after 100 ms
    When SIGTERM arrives
    Then that request is replied to, no new request is taken, and cprime_rt_serve returns 0
    Given a running request that needs 2 s
    When SIGTERM arrives
    Then cprime_rt_serve returns 8 after about 500 ms

  Scenario: Error path — transport fails to start
    Given a transport whose start returns 1 with reason "address in use"
    When cprime_rt_serve runs
    Then stderr is "cprime: transport acme::HttpTransport failed to start: address in use" and it returns 9

#### Performance, Security, Observability

- **Performance targets:** `cprime_rt_alloc` + `cprime_rt_drop` pair ≤ 100 ns p95 for payloads ≤ 64 bytes on the two supported targets (measured by a release-mode micro-benchmark test in `crates/cprime-provider/tests/bench_alloc.rs` over 1,000,000 iterations); `cprime_rt_dup` is one load-add-store; `cprime_rt_drop`, `cprime_rt_eq` are O(number of cells reached) with no recursion; `cprime_rt_concat`, `cprime_rt_push`, `cprime_rt_slice` are O(output size). Per-cell overhead is 24 bytes.
- **Security:** every provider joins the trusted computing base (HLD security posture). Boundary values are copies; providers must not retain argument pointers. The memory limit bounds heap growth of every run; the CPU-time limit bounds release binaries. Trap and limit exits use `_exit` so partial output is never flushed. No input is read from the environment; no file is opened. `unsafe` is confined to `rt` and the `arg_*`/`ret_*` helpers; each `unsafe fn` documents its preconditions (valid header address, live object, correct parameter position).
- **Observability:** this crate writes nothing except the stderr report of mode-0 traps and the CPU-limit action (step 8): one first line, located when a site is known, plus the call chain in call-chain builds. The report contains names and source locations only, never a program value (SA-61). Counters for tests and the leak harness: `RtContext.mem_live`, `AllocTracker.live_objects`, `AllocTracker.live_bytes`.

#### Gaps

None.
