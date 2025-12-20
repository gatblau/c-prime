# Component spec — cprime-codegen

> **Spec set:** cprime
> **Status:** Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> **Last updated:** 2025-12-20

### SPEC: cprime-codegen
**File:** `crates/cprime-codegen/src/lib.rs` (modules `semantics.rs`, `interp.rs`, `jit.rs`, `marshal.rs`, `aot.rs`) | **Package:** `cprime-codegen` | **Phase:** 2 (interpreter), 3 (JIT, marshal), 5 (AOT) | **Dependencies:** `cprime-core`, `cprime-provider`; third-party `cranelift-codegen`, `cranelift-frontend`, `cranelift-module`, `cranelift-jit`, `inkwell` (feature `llvm18-0`)

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprime-codegen` executes and compiles RC-inserted C′-IR in three ways that must agree bit for bit: an **IR interpreter** (the reference oracle for Phase 2 and for differential tests), a **Cranelift JIT** (used by `cprimed` for T1 compilation and by `cprime-verify` for in-memory fuzzing, ADR-008, ADR-013) and an **LLVM AOT back end** (release object files, ADR-008). It also converts C′ values between their JSON-facing form (`Value`) and the one-word run-time form (`marshal`). This spec fixes, in one place, the evaluation order, checked arithmetic and trap behaviour that all three back ends implement (gate item, Phase 2).

#### Approach

One normative semantics table (*Internal Logic*, Part 1) is the contract; each back end is a translation of it, and the runtime functions in `cprime-provider` (`cprime_rt_*`) implement every operation that touches heap memory, so all three back ends share one implementation of allocation, reference counting, equality and byte operations. Arithmetic, comparisons, tag dispatch and field loads are emitted inline. The interpreter uses an explicit frame stack rather than host recursion so that deep C′ recursion cannot overflow the host stack. Provider operations in the JIT are called through a per-module operation table so that `cprime-verify` can replace any operation with a fuzzing stub without recompiling. Alternatives rejected: (a) compiling the interpreter's behaviour from a shared Cranelift path — leaves no independent oracle; (b) per-operation exported symbols for providers in release builds — the runtime package cannot know provider operation names before it is compiled, so release code calls through the exported table `cprime_rt_op_table` instead (§2C).

#### Shared Context

Copied from `phase-2-architecture.md` §2C and §2E; only the items this crate uses.

```rust
// cprime_core::ids
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct Symbol(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct DefId(pub [u8; 32]);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct CapId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct ValueId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct BlockId(pub u32);

// cprime_core::ir
#[derive(Clone, PartialEq, Eq, Hash)]
pub enum Type { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<Type>), Struct(DefId), Sum(DefId),
                Fn { params: Vec<Type>, ret: Box<Type>, effects: BTreeSet<CapId> }, Token }
#[derive(Clone, PartialEq, Eq, Hash)]
pub enum Const { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(Vec<u8>) }
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub enum BinOp { Add, Sub, Mul, Div, Rem, WrappingAdd, WrappingSub, WrappingMul,
                 Eq, Ne, Lt, Le, Gt, Ge, And, Or, BytesConcat }
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub enum IntrinsicOp { U64Be, Push, Slice, ToBytes }
#[derive(Clone)]
pub enum Inst {
    Const(Const), Bin { op: BinOp, lhs: ValueId, rhs: ValueId }, Not(ValueId),
    Call { func: DefId, args: Vec<ValueId> },
    Perform { cap: CapId, op: Symbol, args: Vec<ValueId> },
    Intrinsic { op: IntrinsicOp, args: Vec<ValueId> },
    MakeStruct { ty: DefId, fields: Vec<ValueId>, reuse: Option<ValueId> },
    GetField { value: ValueId, index: u32 },
    MakeVariant { ty: DefId, tag: u32, fields: Vec<ValueId>, reuse: Option<ValueId> },
    GetVariantField { value: ValueId, tag: u32, index: u32 },
    MakeVec { elem: Type, items: Vec<ValueId> },
    Len(ValueId), Index { vec: ValueId, index: ValueId },
    ParMap { func: DefId, xs: ValueId },
    RcDup(ValueId), RcDrop(ValueId), DropReuse(ValueId),
}
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub enum TrapKind { Overflow, DivisionByZero, IndexOutOfBounds, Unreachable }
#[derive(Clone)]
pub enum Terminator {
    Return(ValueId),
    Jump { target: BlockId, args: Vec<ValueId> },
    Branch { cond: ValueId, then_target: BlockId, then_args: Vec<ValueId>, else_target: BlockId, else_args: Vec<ValueId> },
    Switch { scrutinee: ValueId, cases: Vec<(BlockId, Vec<ValueId>)> },
    Trap(TrapKind),
}
#[derive(Clone)] pub struct Block { pub params: Vec<(ValueId, Type)>, pub insts: Vec<(ValueId, Inst)>, pub term: Terminator }
#[derive(Clone)] pub struct Function { pub def: DefId, pub ty: Type, pub entry: BlockId, pub blocks: Vec<Block>, pub value_count: u32,
                                       pub node_map: BTreeMap<ValueId, NodeId> }  // (D3, D23) source node of every parameter, non-constant instruction, join parameter and Return value; read here only by AOT for trap sites (D27) and by the step 27 tests
pub struct Layouts { pub structs: BTreeMap<DefId, Vec<Type>>, pub sums: BTreeMap<DefId, Vec<Vec<Type>>> }

// cprime_core::value
#[derive(Clone, PartialEq, Eq)]
pub enum Value { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(Vec<u8>), FixedBytes(Vec<u8>), Vec(Vec<Value>),
                 Struct { fields: Vec<(String, Value)> }, Variant { variant: String, fields: Vec<(String, Value)> }, Str(String) }
pub struct TypeSchema {
    pub structs: BTreeMap<DefId, Vec<(String, Type)>>,
    pub sums: BTreeMap<DefId, Vec<(String, Vec<(String, Type)>)>>,
}

// cprime_core::program
pub struct Program {
    pub functions: BTreeMap<DefId, ir::Function>,   // after RC insertion
    pub layouts: Layouts,
    pub schema: TypeSchema,
    pub borrows: BTreeMap<DefId, BorrowSig>,
    pub op_targets: BTreeMap<(CapId, Symbol), OpTarget>,
    pub unimplemented: BTreeSet<DefId>,             // calls link to a trap stub (Unreachable)
}
#[derive(Clone, PartialEq, Eq)] pub struct BorrowSig { pub borrowed: Vec<bool> }
#[derive(Clone, Copy, PartialEq, Eq)] pub enum OpTarget { Provider { flat_index: u32 }, Stub }

// cprime_provider
pub type OpFn = unsafe extern "C" fn(args: *const u64, rt: *mut RtContext) -> u64;
pub struct OpDecl { pub name: &'static str, pub params: Vec<BoundaryType>, pub ret: BoundaryType, pub deterministic: bool, pub func: OpFn,
                   pub vectors: &'static [KnownAnswer] }   // (D40) not read by this crate
pub struct ProviderDecl { pub path: &'static str, pub crate_name: &'static str, pub crate_version: &'static str, pub ops: Vec<OpDecl>, pub thread_safe: bool, pub transport: Option<TransportDecl> }
pub struct ProviderRegistry { pub providers: Vec<ProviderDecl> }
impl ProviderRegistry { pub fn flat_ops(&self) -> Vec<(&ProviderDecl, &OpDecl)>; }

#[repr(C)]
pub struct RtContext {      // field order is ABI: compiled code addresses fields by these offsets
    pub trap: u32,          // offset 0
    pub mode: u32,          // offset 4
    pub fuel: u64,          // offset 8
    pub depth: u64,         // offset 16
    pub max_depth: u64,     // offset 24
    pub mem_limit: u64,     // offset 32
    pub mem_live: u64,      // offset 40
    pub tracker: *mut AllocTracker,   // offset 48
    pub site: u64,          // offset 56: site id of the Perform in progress (AOT only, D27); 0 = none
    pub chain: *mut u32,    // offset 64: call-chain buffer, max_depth + 1 entries (D28); null = no call chain
}
#[repr(u32)] #[derive(Clone, Copy, PartialEq, Eq)]
pub enum RtTrap { None = 0, Overflow = 1, DivisionByZero = 2, IndexOutOfBounds = 3, Unreachable = 4,
                  FuelExhausted = 5, MemoryLimit = 6, ProviderPanic = 7, StackLimit = 8 }
```

The interpreter and the JIT never read or write `site` or `chain`; every context they receive has `site: 0` and `chain: null`.

Runtime functions (owned by `cprime-provider`, `#[no_mangle] extern "C"`): `cprime_rt_alloc(rt, payload_bytes, len, tag, kind, ptr_mask) -> u64`, `cprime_rt_dup(ptr)`, `cprime_rt_drop(rt, ptr)`, `cprime_rt_drop_reuse(rt, ptr) -> u64`, `cprime_rt_eq(a, b) -> u64`, `cprime_rt_concat(rt, a, b) -> u64`, `cprime_rt_u64_be(rt, x) -> u64`, `cprime_rt_push(rt, v, x) -> u64`, `cprime_rt_slice(rt, b, start, end) -> u64`, `cprime_rt_to_bytes(rt, b) -> u64`, `cprime_rt_trap(rt, code)`, `cprime_rt_trap_at(rt, code: u32, site: u32)` (D27: in mode 0 it reports the location of `site` from the trap site table and exits; in mode 1 it behaves as `cprime_rt_trap`). The trap site table and function table are registered by the generated release main through `cprime_provider::rt::set_report_tables(sites: &'static [SiteEntry], functions: &'static [SiteEntry])`; this crate emits only the ids that index them. The generated release runtime (`components/cprime-cli.md`) exports `#[no_mangle] pub static cprime_rt_op_table: AtomicPtr<OpFn>`, which its `init_op_table()` sets, before the entry runs, to a heap array of every provider `OpFn` in `flat_ops()` order (D7).

Heap object header (bytes): `[0..8) rc: u64`, `[8..12) len: u32`, `[12..14) tag: u16`, `[14] kind: u8` (0 bytes, 1 Vec, 2 struct, 3 variant), `[15]` reserved, `[16..24) ptr_mask: u64`, payload from offset 24 (bytes, or 8-byte words).

Runtime protocols 1–7 of §2E are binding; the ones this crate implements are restated where used in *Internal Logic*. Release exit codes: 0 success, 2 bad input, 3 trap, 4 CPU-time limit, 5 memory limit, 6 provider panic, 7 call-depth limit (`StackLimit`).

Configuration keys read: `limits.cpu_time_ms`, `limits.memory_bytes` (passed through by `cprime-cli` to the release main, not read here directly); `limits.max_call_depth` (u64, 1,000 … 10,000,000, default 100000; passed through by `cprime-cli`, sets `rt.max_depth`).

**Parallel map (RFC 02, D18, D19).** `Inst::ParMap { func, xs }` lowers in every back end to `cprime_rt_par_map(rt, f, owns_arg, ret_is_ptr, xs)`: `f` is the address of the compiled `func` (or of its trap stub when `func` is in `program.unimplemented`), `owns_arg = 1` when `program.borrows[func]` marks parameter 0 owned (else 0), `ret_is_ptr = 1` when `ret(func)` is heap-typed. The interpreter executes the same sequential loop as `cprime_rt_par_map`'s sequential path on its explicit frame stack (no host recursion), with fuel and depth per step 4. JIT and AOT emit the call; in mode 1 it is followed by the `rt.trap` test of step 6. Release binaries take the parallel path when `rt.mode == 0`.

**Witness traces and release trap locations (RFC 03, D23, D24, D27, D28).**
- **D23:** lowering gives `node_map` an entry for every entry-block parameter, every non-constant instruction and every join-block parameter. This crate reads it only to fill `AotSite.node`.
- **D24:** `interp::replay` runs a function like `interp::run`. It asks a caller-supplied `ReplayOracle` before every `Call`, `Perform` and `ParMap`, and records the called function's own activation as `ReplayEvent`s (Part 2a). A trap inside a callee or a `par_map` element adds one depth-1 `Trap { at: Some(<the Call or ParMap in progress>), kind }` event, while `ReplayOutcome::Trapped` still names the trapping function and its own `at`. Result types of `Call` and `Perform` instructions, which C′-IR leaves opaque, come first from the oracle (`call_type`, `op_type`), then from the program's own evidence. Replay stops with `Mismatch` only when both oracle and program give no type, so a caller that supplies complete types never meets it. `run` is unchanged, and a stubbed operation in `run` still traps `Unreachable` (SA-42).
- **D27:** with `AotOptions.trap_sites` (always `true` from `cprime-cli`; `false` only for the Phase 5 no-reporting benchmark baseline), AOT code passes a constant site id on every trap edge (`cprime_rt_trap_at`). The prologue's depth check passes its `Entry` site. Every `Perform` stores its site id in `rt.site` before the call. `Intrinsic Slice` is preceded by an inline range check with its own site. `emit_object` returns the site list and the function list, which `cprime-cli` turns into the tables of the release binary.
- **D28:** with `AotOptions.call_chain`, every function's prologue writes its function index to `rt.chain[rt.depth]` after the depth check.

#### Public Interface

```rust
// ---- errors (crates/cprime-codegen/src/lib.rs) ----
#[derive(Debug)]
pub enum CodegenError {
    MissingFunction { def: DefId },                                  // Call names a DefId absent from functions and unimplemented
    MissingOpTarget { cap: CapId, op: Symbol },                      // Perform with no op_targets entry
    ProviderIndexOutOfRange { flat_index: u32, available: u32 },
    UnboundOperation { cap: CapId, op: Symbol },                     // AOT only: OpTarget::Stub reached
    UnimplementedCallee { def: DefId },                              // AOT only: unimplemented non-empty
    NotAnOperation { cap: CapId, op: Symbol },                       // set_op on an operation the module does not perform
    Cranelift { def: DefId, message: String },
    Llvm { message: String },
    TargetUnsupported { triple: String },                            // host not one of the two SA-03 targets
    Io { path: PathBuf, message: String },
}

// ---- interpreter (crates/cprime-codegen/src/interp.rs) ----
pub fn run(program: &Program, func: DefId, args: &[Value], rt: &mut RtContext,
           registry: &ProviderRegistry) -> Result<Value, RtTrap>;

/// (D24) Runs `func` like `run`, asking `oracle` before every Call, Perform and ParMap, and records the
/// events of `func`'s own activation (depth 1). Never panics on an oracle answer; see Part 2a.
pub fn replay(program: &Program, func: DefId, args: &[Value], rt: &mut RtContext,
              registry: &ProviderRegistry, oracle: &mut dyn ReplayOracle) -> Replay;

/// Supplies, or declines to supply, the result of a call, operation or parallel map during `replay`.
/// `depth` is the activation depth: 1 = the replayed function's own activation.
pub trait ReplayOracle {
    fn call(&mut self, at: ValueId, depth: u64, callee: DefId, args: &[Value]) -> Answer;
    fn perform(&mut self, at: ValueId, depth: u64, cap: CapId, op: Symbol, args: &[Value]) -> Answer;
    fn par_map(&mut self, at: ValueId, depth: u64, func: DefId, xs: &Value) -> Answer;
    /// Result type of a call to `callee`, when the oracle knows it (cprime-verify: from the callee's TSpec).
    fn call_type(&self, callee: DefId) -> Option<Type>;
    /// Result type of operation `op` of capability `cap`, when the oracle knows it (cprime-verify: from CapOpSig).
    fn op_type(&self, cap: CapId, op: Symbol) -> Option<Type>;
}
pub enum Answer {
    Value(Value),   // use this value as the result; the callee, operation or mapped spec is not executed
    Live,           // execute exactly as `run` does
    Mismatch,       // stop the replay with ReplayOutcome::Mismatch { at }
}
pub struct Replay { pub outcome: ReplayOutcome, pub events: Vec<ReplayEvent> }
pub enum ReplayOutcome {
    Returned(Value),
    Trapped { kind: RtTrap, func: DefId, at: Option<ValueId> },   // func: the function that trapped (any depth); at: None for a terminator or prologue trap
    Mismatch { at: ValueId },
}
pub enum ReplayEvent {                                            // depth-1 activation only, in execution order
    Inst { at: ValueId, value: Value, answered: bool },            // answered: the value came from Answer::Value
    Term { block: BlockId, taken: u32 },                           // taken: 0 for Jump; 0 = then, 1 = else for Branch; the tag for Switch; 0 for Return
    Trap { at: Option<ValueId>, kind: RtTrap },                    // a trap deeper down: at = the depth-1 Call or ParMap in progress
}

// ---- JIT (crates/cprime-codegen/src/jit.rs) ----
pub struct JitModule { /* Cranelift JITModule, function pointer map, operation table */ }
impl JitModule {
    pub fn compile(program: &Program, registry: &ProviderRegistry) -> Result<JitModule, CodegenError>;
    /// Calls a compiled function with word arguments. Safety: `args.len()` and each word's
    /// representation must match the function's parameter types; `rt` must be valid for the call;
    /// the calling thread's stack must be at least `rt.max_depth × 1 KiB`.
    pub unsafe fn call(&self, func: DefId, args: &[u64], rt: &mut RtContext) -> u64;
    /// Operations the compiled code performs, with their initial targets (§2E, D4).
    pub fn ops(&self) -> Vec<(CapId, Symbol, OpTarget)>;
    /// Replaces the function called for one operation (§2E, D4). Used by cprime-verify to install
    /// fuzzing stubs, memoising stubs and the fuzz_live wrapper (SA-30, D42); `op_id` is passed back to `f` on every call so one
    /// stub function can serve many operations. Takes effect for subsequent calls.
    /// Takes `&self` so the T3 provider thread that owns the module (D41) can install stubs and the
    /// `fuzz_live` wrapper (D42) while calling it; the module is `Send`, never `Sync`.
    pub fn set_op(&self, cap: CapId, op: Symbol, f: StubFn, op_id: u32) -> Result<(), CodegenError>;
    /// Restores every operation to its initial target (§2E, D4).
    pub fn reset_ops(&self);
}

/// Shape of every JIT operation-table slot (§2E, D4).
pub type StubFn = unsafe extern "C" fn(op_id: u32, args: *const u64, rt: *mut RtContext) -> u64;

// ---- marshal (crates/cprime-codegen/src/marshal.rs) ----
pub fn value_to_word(v: &Value, ty: &Type, program: &Program, rt: &mut RtContext) -> u64;
pub fn word_to_value(w: u64, ty: &Type, program: &Program) -> Value;

// ---- AOT (crates/cprime-codegen/src/aot.rs) ----
pub struct AotOutput {
    pub object: PathBuf,
    pub entry_symbol: String,          // always "cprime_entry"
    pub sites: Vec<AotSite>,           // (D27) site id = index + 1; 0 is never a site id
    pub functions: Vec<DefId>,         // (D27, D28) function index = position + 1, ascending DefId
}
pub struct AotSite { pub func: DefId, pub node: NodeId, pub kind: AotSiteKind }
pub enum AotSiteKind { Trap(TrapKind), Entry, Perform { cap: CapId, op: Symbol } }
pub struct AotOptions {
    pub call_chain: bool,   // (D28) from configuration key release.call_chain
    pub trap_sites: bool,   // (D27) default true; never read from configuration; cprime-cli always passes true;
                            // only the Phase 5 benchmark harness passes false, to build the no-reporting baseline
}
impl Default for AotOptions { fn default() -> Self { AotOptions { call_chain: false, trap_sites: true } } }
pub fn emit_object(program: &Program, entry: DefId, out: &Path, opts: &AotOptions) -> Result<AotOutput, CodegenError>;
```

`NodeId` is `cprime_core::ids::NodeId(pub u32)`: the pre-order index of a node in the function's `.opq` source (SA-32).

Preconditions shared by every entry point: each `Function` in `program.functions` verifies at `IrStage::RcInserted` (the caller, `cprime-daemon` or `cprime-cli`, has run `cprime-memory`). `value_to_word` and `word_to_value` require `v` to match `ty`; a mismatch is a caller defect and panics with a message naming the type (library panic permitted only here because the signature has no error channel; callers construct values from `TypeSchema`-checked JSON or from the verifier's own generator).

##### Example

Program: `Auth::generate_token` after RC insertion (see `cprime-memory` Example), `CryptoCap::hmac_sha256` → `OpTarget::Provider { flat_index: 1 }` (`host::Crypto` is first in the registry, `hmac_sha256` its second operation).

```text
interp::run(program, generate_token, [U64(1), U64(100), Bytes(0x00…1f /* 32 bytes */)], rt{mode:1, fuel:1000, max_depth:100000}, registry)
  → Ok(Struct { fields: [("user_id", U64(1)), ("expires_at", U64(3700)), ("sig", FixedBytes(<32 bytes HMAC>))] })

interp::run(program, generate_token, [U64(1), U64(18446744073709551615), Bytes(…32 bytes…)], …)
  → Err(RtTrap::Overflow)                    ; v4 = Add v1 3600 overflows

JitModule::compile(program, registry)?.call(generate_token, [1, 100, ptr_to_secret], rt)
  → pointer to a struct cell; word_to_value(…) equals the interpreter's Ok value above

emit_object(program, generate_token, ".cprime/release/obj/cprime_entry.o", &AotOptions { call_chain: false, trap_sites: true })
  → Ok(AotOutput { object: ".cprime/release/obj/cprime_entry.o", entry_symbol: "cprime_entry",
                   sites: [ AotSite { func: generate_token, node: NodeId(0),  kind: Entry },                          ; site 1
                            AotSite { func: generate_token, node: NodeId(6),  kind: Trap(Overflow) },                 ; site 2: v4 = Add v1 v3
                            AotSite { func: generate_token, node: NodeId(16), kind: Perform { cap: CapId(1), op: hmac_sha256 } } ],  ; site 3: v8
                   functions: [generate_token] })
  ; the overflow edge of v4 calls cprime_rt_trap_at(rt, 1, 2); before v8, AOT code stores 3 in rt.site

interp::replay(program, generate_token, [U64(1), U64(100), Bytes(…32 bytes…)], rt{mode:1, fuel:1000, max_depth:100000}, registry, oracle)
  ; oracle.perform(at: v8, depth: 1, CapId(1), hmac_sha256, [Bytes(secret), Bytes(msg)]) answers Value(FixedBytes(0x00…00))
  → Replay { outcome: Returned(Struct { fields: [("user_id", U64(1)), ("expires_at", U64(3700)), ("sig", FixedBytes(0x00…00))] }),
             events: [ Inst { at: v3, value: U64(3600), answered: false }, Inst { at: v4, value: U64(3700), answered: false },
                       Inst { at: v5, … }, Inst { at: v6, … }, Inst { at: v7, … },
                       Inst { at: v8, value: FixedBytes(0x00…00), answered: true },
                       Inst { at: v9, value: Struct { … }, answered: false },
                       Term { block: BlockId(0), taken: 0 } ] }
  ; RcDup/RcDrop/DropReuse inserted by cprime-memory produce no event
```

#### Internal Logic

**Part 1 — Normative semantics (all three back ends)**

1. **Words.** Every value is one 64-bit word (SA-25): `Unit` = 0; `Bool` = 0 or 1; `U8` zero-extended; `U64` as is; `I64` two's complement; heap values = address of the header; `Token` = address or 0. Back ends never read the upper 56 bits of a `U8` or the upper 63 bits of a `Bool` as significant; producers always write them as 0.
2. **Evaluation order.** Instructions of a block execute in list order, then the terminator. Back ends may reorder only instructions with no observable effect and no trap: `Const`, `Not`, `Bin` with `WrappingAdd`/`WrappingSub`/`WrappingMul`/`Eq`/`Ne`/`Lt`/`Le`/`Gt`/`Ge`/`And`/`Or` on scalars, `GetField`, `GetVariantField`, `Len`. All other instructions (trap sites, calls, performs, allocations, RC instructions, heap `Eq`/`Ne`) keep their relative order. Argument evaluation order is the IR's order; `cprime-types` lowering emits it left to right.
3. **Semantics table** (normative):

| Instruction | Operand types | Result | Trap / effect |
|---|---|---|---|
| `Const c` | — | the constant; `Bytes` constants allocate a fresh `kind 0` object | `MemoryLimit` if allocation fails |
| `Bin Add/Sub/Mul` | both `U8`, both `U64` or both `I64` | exact result | `Overflow` if outside the type's range (`U8` 0…255, `U64` 0…2^64−1, `I64` −2^63…2^63−1) |
| `Bin Div` | same as above | quotient truncated toward zero | `DivisionByZero` if rhs = 0 (checked first); `Overflow` if `I64` and lhs = −2^63 and rhs = −1 |
| `Bin Rem` | same as above | remainder with the sign of the dividend (Rust `%`) | `DivisionByZero` if rhs = 0; `I64::MIN % -1` = 0, no trap |
| `Bin WrappingAdd/Sub/Mul` | same as above | result modulo 2^8 (`U8`) or 2^64 (`U64`, `I64` two's complement) | none |
| `Bin Lt/Le/Gt/Ge` | `U8`/`U64` unsigned, `I64` signed | `Bool` | none |
| `Bin Eq/Ne` | any equal types except `Fn`, `Token` | `Bool`; scalars by word; `Unit` always equal; heap values via `cprime_rt_eq(a, b)` (structural: kind, len, tag, then payload bytes or words, recursing on words whose `ptr_mask` bit is set) | none |
| `Bin And/Or` | `Bool` | `Bool` (strict; short-circuit is lowered to branches) | none |
| `Bin BytesConcat` | `Bytes`, `Bytes` | `cprime_rt_concat(rt, a, b)` | `MemoryLimit` |
| `Not v` | `Bool` | `1 − v` | none |
| `Call f args` | per callee signature | callee result | callee traps propagate (step 6) |
| `Perform cap op args` | per operation signature | operation result | provider panic → `ProviderPanic`; stub (interpreter `run`) → `Unreachable`; in `replay` the oracle is asked first (step 12b) |
| `Intrinsic U64Be [x]` | `U64` | 8-byte big-endian `Bytes` via `cprime_rt_u64_be` | `MemoryLimit` |
| `Intrinsic Push [v, x]` | `Vec[T]`, `T` | new `Vec` with `x` appended via `cprime_rt_push` (borrows `v` and `x`; duplicates `x` when heap) | `MemoryLimit` |
| `Intrinsic Slice [b, s, e]` | `Bytes`, `U64`, `U64` | bytes `[s, e)` via `cprime_rt_slice` | `IndexOutOfBounds` unless `s ≤ e ≤ len(b)`; AOT with `trap_sites` checks the range inline before the call, so the runtime's own check is unreachable there (step 24) |
| `Intrinsic ToBytes [b]` | `FixedBytes[N]` | `Bytes` copy via `cprime_rt_to_bytes` | `MemoryLimit` |
| `MakeStruct ty fields reuse` | field types of `ty` | cell with `kind 2`, `len = fields.len()`, `tag 0`, `ptr_mask` from heap-typed field positions | `MemoryLimit` when allocating |
| `MakeVariant ty tag fields reuse` | field types of variant `tag` | cell with `kind 3`, `tag`, `len`, `ptr_mask` | `MemoryLimit` when allocating |
| `MakeVec elem items` | `elem` | cell with `kind 1`, `len = items.len()`, `ptr_mask` bit 0 set iff `elem` is heap-typed | `MemoryLimit` |
| `GetField v i` / `GetVariantField v tag i` | struct / variant with that tag | payload word `i` (borrowed) | none (tag guaranteed by the IR verifier) |
| `Len v` | `Bytes`, `FixedBytes`, `Vec` | header `len` zero-extended | none |
| `Index v i` | `Vec[T]` → word `i`; `Bytes`/`FixedBytes` → byte `i` zero-extended | element (borrowed when heap) | `IndexOutOfBounds` if `i ≥ len` |
| `ParMap f xs` | `Vec[T]`, `f : T → U` | `Vec[U]`: `f` applied to each element in index order, via `cprime_rt_par_map(rt, f, owns_arg, ret_is_ptr, xs)` (D19) | the trap of the lowest-index element propagates |
| `RcDup v` | heap | `rc += 1` (may be inlined) | none |
| `RcDrop v` | heap | `cprime_rt_drop(rt, v)` | none |
| `DropReuse v` | heap | `cprime_rt_drop_reuse(rt, v)` → token | none |
| `Return v` | — | function result | — |
| `Jump`, `Branch` | `Bool` cond (1 = then) | transfer with block arguments | — |
| `Switch v cases` | sum | jump to `cases[tag(v)]` | a tag ≥ `cases.len()` cannot occur (IR verifier); back ends emit `Unreachable` for it |
| `Trap k` | — | — | trap `k` |

   For a `Make*` with `reuse: Some(t)`: if `t ≠ 0` the cell at `t` is overwritten (header `rc = 1`, new `len`, `tag`, `kind`, `ptr_mask`; payload words written in field order) and no allocation occurs; if `t = 0`, `cprime_rt_alloc` is called. When a back end inlines `RcDup`, it performs the same non-atomic increment the runtime function performs.
4. **Function prologue** (every compiled function, all back ends, in this order): (a) if `rt.fuel ≠ u64::MAX`: if `rt.fuel = 0` trap `FuelExhausted`, else `rt.fuel −= 1`; (b) `rt.depth += 1`; if `rt.depth > rt.max_depth` trap `StackLimit`. Epilogue on every return path: `rt.depth −= 1`. With `fuel = N`, exactly N function entries succeed and entry N + 1 traps.
5. **Trapping** (protocol 3). A trap calls `cprime_rt_trap(rt, code)`. In mode 0 that function does not return (it exits with the release exit code). In mode 1 it records `rt.trap = code` (first trap wins: a non-zero `rt.trap` is not overwritten) and returns; the trapping function then returns word 0 immediately. AOT code built with `trap_sites` calls `cprime_rt_trap_at(rt, code, site)` instead on every trap edge it emits (D27, step 24); the interpreter and the JIT always call `cprime_rt_trap`.
6. **Propagation** (mode 1). After every `Call`, `Perform`, runtime-function call that can trap (`cprime_rt_alloc`, `cprime_rt_par_map`, `cprime_rt_concat`, `cprime_rt_u64_be`, `cprime_rt_push`, `cprime_rt_slice`, `cprime_rt_to_bytes`) and inline trap-site check, generated code tests `rt.trap ≠ 0` and, when set, returns 0. No RC cleanup happens on this path; callers in mode 1 discard the run and release memory with `AllocTracker::free_all` (protocol 7). The AOT back end, which always runs in mode 0, omits these tests, because in mode 0 no trap returns.
7. **Perform.** Arguments are written to a contiguous array of words in argument order; the operation is called as `OpFn(args_ptr, rt)`; arguments are borrowed (protocol 5); the result is owned. Target selection: interpreter — `OpTarget::Provider { flat_index }` calls `registry.flat_ops()[flat_index].1.func`; `OpTarget::Stub` traps `Unreachable`. JIT — the operation table slot, called as `StubFn(op_id, args_ptr, rt)` (Part 3). AOT — `Provider` loads the array pointer from the global `cprime_rt_op_table` (`AtomicPtr<OpFn>`, relaxed load) and calls entry `flat_index` as `OpFn(args_ptr, rt)`; `Stub` is a `CodegenError::UnboundOperation`.
8. **Calls to unimplemented specs.** For every `DefId` in `program.unimplemented`, the interpreter and JIT bind the callee to a stub with the callee's parameter count that traps `Unreachable` (after the prologue of step 4). AOT returns `CodegenError::UnimplementedCallee` when `program.unimplemented` is non-empty.

**Part 2 — Interpreter (`interp::run`)**

9. **Validate inputs.** Look up `func` in `program.functions`; if absent, log `error` `codegen.interp_bad_call` (`def`) and return `Err(RtTrap::Unreachable)`. Check `args.len()` equals the entry block parameter count; on mismatch the same event and result.
10. **Marshal arguments** with `marshal::value_to_word` against the entry block parameter types. Arguments are owned by the callee according to `program.borrows[func]`: for borrowed parameters the interpreter keeps its reference and drops it with `cprime_rt_drop` after the call returns (or leaves it to `free_all` on trap).
11. **Execute** with an explicit frame stack (`Vec<Frame>`, `Frame { def, block, inst_index, regs: Vec<u64> }`, `regs` sized `value_count`) — no host recursion, so C′ recursion depth is bounded only by `rt.max_depth`. Each instruction follows Part 1 exactly, calling the same `cprime_rt_*` functions the compiled back ends call (they are ordinary Rust functions in `cprime-provider`). Fuel and depth follow step 4. `rt.mode` must be 1; if it is 0 the interpreter sets it to 1 for the run and restores it afterwards (the interpreter never exits the host).
11a. **Pre-RC programs** (D45). `run` does not check the IR stage. A `Program` whose functions are at `IrStage::Lowered` (no `RcDup`, `RcDrop`, `DropReuse`) executes with the semantics of Part 1; nothing is released during the run, so heap values remain live until the caller calls `AllocTracker::free_all`, and the memory limit counts every allocation. Only the Part 6 item 29 test runs such programs.
12. **Finish.** On `Return` from the outermost frame, convert the result with `marshal::word_to_value`, drop the result word with `cprime_rt_drop` when heap-typed, and return `Ok(value)`. On a trap, return `Err(code as RtTrap)` with `rt.trap` left set. Log `trace` event `codegen.interp_run` with `def`, `steps` (u64), `result` (`ok` or trap name).

**Part 2a — Replay (`interp::replay`, D24)**

12a. **Validate and marshal** as steps 9 and 10. An unknown function or a wrong argument count returns `Replay { outcome: Trapped { kind: Unreachable, func, at: None }, events: [] }` and logs `codegen.interp_bad_call`.
12b. **Execute** exactly as step 11, on the same frame stack, with these differences:
   - **Value types.** Replay keeps a type for every value of every frame. The types come from:
     - block parameters (declared);
     - the result types of the `operand-types` rule of `cprime-core`'s IR verifier, for every non-opaque instruction.

     For the results of `Call` and `Perform`, which that rule leaves opaque, replay asks the oracle first:
     - `oracle.call_type(callee)` for a `Call`;
     - `oracle.op_type(cap, op)` for a `Perform`.

     When the oracle returns `None`, replay falls back to the program's own evidence:
     - for a `Call`, the callee's `Function.ty` when the callee is in `program.functions`;
     - for a `Perform` whose target is `OpTarget::Provider`, the bound operation's `OpDecl` result type (`Unit`, `Bool`, `U8`, `U64`, `I64`, `Bytes`, `FixedBytes[N]`, `Vec[T]` map to the `Type` of the same name; `Result[T, E]` gives no type here);
     - for a `ParMap`, `Vec(ret(func))`, with `ret(func)` from `program.functions[func].ty`.

     A value for which both oracle and program give no type is *untyped*. Arguments are values defined earlier, so they take the types of their definitions.
   - **Oracle requests.** Before a `Call`, `Perform` or `ParMap` at activation depth *d* (1 = the replayed function), convert the arguments with `marshal::word_to_value` and ask the oracle: `call(at, d, callee, args)`, `perform(at, d, cap, op, args)` or `par_map(at, d, func, xs)`. An untyped argument or result makes the request impossible, and replay stops with `Mismatch { at }`. This is reachable only through an oracle that omits a type the program cannot supply. Callers that return complete types from `call_type` and `op_type` never meet it: `cprime-verify` does, from callee `TSpec`s and `CapOpSig`s.
   - **`Answer::Value(v)`.** Do not execute the callee, operation or mapped spec. Release the arguments exactly as the call would have: owned arguments (per `program.borrows` for `Call` and `ParMap`) are dropped with `cprime_rt_drop`, and borrowed arguments are left alone. Then write `marshal::value_to_word(v, result type)` to the result register. A `v` that does not match the result type is a caller defect: a debug assertion fails, and in release builds the replay stops with `Mismatch { at }`.
   - **`Answer::Live`.** Execute exactly as step 11. A `Perform` with target `Stub` that is answered `Live` traps `Unreachable`, as in `run`.
   - **`Answer::Mismatch`.** Stop with `Mismatch { at }`.
12c. **Record events** for the depth-1 activation only, in execution order:
   - `Inst { at, value, answered }` for every instruction except `RcDup`, `RcDrop` and `DropReuse`, after it completes. `value` is `word_to_value` of the result; `answered` is true iff it came from `Answer::Value`. An instruction whose result is untyped stops the replay with `Mismatch { at }`.
   - `Term { block, taken }` for every terminator: `taken` is 0 for `Jump` and `Return`, 0 (then) or 1 (else) for `Branch`, and the tag for `Switch`.
   - `Trap { at, kind }` when the depth-1 activation traps. `at` is the trapping instruction, or `None` for a `Trap` terminator and for a prologue trap.
   - `Trap { at: Some(<call>), kind }` when a trap happens deeper: inside a callee (depth > 1) or inside a `par_map` element. `<call>` is the `ValueId` of the depth-1 `Call` or `ParMap` instruction in progress. It is the depth-1 activation's last event. `ReplayOutcome::Trapped` still reports the function that trapped and that function's own `at` (step 12d).

   Events of deeper activations are not recorded.
12d. **Finish.**
   - On `Return` from the outermost frame: `outcome = Returned(word_to_value(result))`, then drop the result word as step 12 does.
   - On a trap at any depth: `outcome = Trapped { kind, func, at }`, where `func` is the function whose frame trapped and `at` the trapping instruction (`None` for a terminator or prologue trap).
   - On a mismatch: `Mismatch { at }`.

   - Memory: on `Returned`, the RC instructions of the IR have released every other value, as in `run`. On `Trapped` and `Mismatch`, the caller releases the run's memory with `AllocTracker::free_all`, as after a trapped `run` (protocol 7).

   With an `AllocTracker` attached, `live_objects` is therefore 0 after every replay once the result is dropped or `free_all` has run. The mode rules of step 11 apply. Log `trace` event `codegen.interp_replay` with `def`, `steps`, `answered` (count) and `outcome` (`returned`, the trap name or `mismatch`).

**Part 3 — JIT (`JitModule::compile`, `call`, `ops`, `set_op`, `reset_ops`)**

13. **Check host target.** If the host triple is not `x86_64-unknown-linux-gnu` or `aarch64-apple-darwin`, return `CodegenError::TargetUnsupported`.
14. **Collect operations.** For every `Perform` in every function, look up `(cap, op)` in `program.op_targets`; absent → `CodegenError::MissingOpTarget`. For `Provider { flat_index }`, check `flat_index < registry.flat_ops().len()`, else `ProviderIndexOutOfRange`. Allocate one operation-table slot per distinct `(cap, op)` in ascending `(CapId, Symbol)` order. The table is a heap array of `Slot { f: StubFn, op_id: u32 }` owned by the `JitModule`, at a fixed address for the module's lifetime. Initial contents: for a `Provider { flat_index }` target, a trampoline generated in the same Cranelift module (one per provider operation used) that ignores `op_id` and calls the provider's `OpFn(args, rt)`, with `op_id = flat_index`; for a `Stub` target, a built-in `StubFn` that calls `cprime_rt_trap(rt, Unreachable)` and returns 0, with `op_id = 0`.
15. **Declare symbols.** Register each `cprime_rt_*` runtime function by name with its address. Declare every function in `program.functions` and one trap stub per `DefId` in `program.unimplemented`; a `Call` or `ParMap` naming a `DefId` in neither set → `CodegenError::MissingFunction`.
16. **Translate each function** (ascending `DefId`) to Cranelift IR with the platform default C calling convention: parameters `(rt: pointer, a0 … an−1: i64)`, one `i64` result (protocol 1). Each IR block becomes a Cranelift block whose parameters are the IR block parameters (block-argument SSA maps one to one, ADR-011). Emit the prologue of step 4, then each instruction per Part 1, with inline overflow checks (checked add/sub/mul via overflow-flag results, division guards before the divide instruction so that no hardware division fault occurs) branching to a per-function trap block per `TrapKind`; `Perform` loads the slot's `f` and `op_id` from the operation table and calls `f(op_id, args_ptr, rt)` indirectly; propagation tests per step 6. Translation failure → `CodegenError::Cranelift { def, message }`.
17. **Finalise** the module, record the address of every function, and return the `JitModule`. Log `debug` `codegen.jit_compiled` with `functions` (u32), `insts` (u64), `elapsed_us`.
18. **`call`.** Look up the function address (absent → debug assertion; release builds return 0 with `rt.trap = Unreachable`), transmute to the C signature for `args.len()` parameters (supported arities 0…16; C′ specs with more parameters are called through a generated trampoline that reads arguments from an array — the compiler emits one trampoline per function with more than 16 parameters), call it, return the word.
19. **`ops`, `set_op`, `reset_ops`.** `ops` returns the slots in table order with their initial targets. `set_op` writes `(f, op_id)` into the slot of `(cap, op)`; a pair with no slot → `CodegenError::NotAnOperation`. `reset_ops` rewrites every slot to its initial content. Slots are interior-mutable (`Cell<Slot>`), so writes take `&self`; `JitModule` is `Send` and `!Sync` (D41): it may be moved to another thread (`cprime-verify` moves it into its T3 provider thread) but never shared, so no call can be running on another thread during a write, and a write takes effect for the next `call`.

**Part 4 — Marshal**

20. **`value_to_word`.** Scalars: per step 1. `Bytes`/`FixedBytes`: `cprime_rt_alloc(rt, n, n, 0, 0, 0)` then copy bytes. `Vec[T]`: convert each element (heap elements owned by the new vector), allocate `kind 1`. `Struct(d)`: fields in `program.schema.structs[d]` order; field names in the `Value` must equal the schema's in order. `Sum(d)`: tag = position of the variant name in `program.schema.sums[d]`; fields likewise. The returned word owns one reference.
21. **`word_to_value`.** Inverse of step 20 reading memory without changing any count; `FixedBytes(n)` yields `Value::FixedBytes`, `Bytes` yields `Value::Bytes`; struct and variant field names come from the schema.

**Part 5 — AOT (`emit_object`)**

22. **Check inputs.** Host triple as in step 13. `entry` must be in `program.functions` (else `MissingFunction`); `program.unimplemented` must be empty (else `UnimplementedCallee`); every `op_targets` value must be `Provider` (else `UnboundOperation` for the first `Stub` in ascending key order).
23. **Create the LLVM module** for the host triple with relocation model PIC and the target's generic CPU (`x86-64` baseline for Linux, `apple-m1` for macOS arm64), so the binary runs on any machine of that target. Declare the runtime functions and the external global `cprime_rt_op_table` (a pointer-sized global holding the address of the operation array).
24. **Translate each function** with internal linkage, named `cprime_fn_` + the first 16 hex characters of its `DefId` (collisions within one program → append `_` + the next 16 characters until unique), C calling convention, per Part 1 and step 4. Overflow checks use the LLVM overflow-reporting arithmetic intrinsics; no `nsw`/`nuw`/`exact` flags are emitted, and division is guarded before execution, so the module contains no undefined behaviour that the optimiser could exploit. Mode-1 propagation tests are omitted (step 6). `Perform` loads the array address from `cprime_rt_op_table`, then loads entry `flat_index` from that array and calls it indirectly.
24a. **Check options.** `opts.call_chain && !opts.trap_sites` → `CodegenError::Llvm { message: "call_chain requires trap_sites" }`, before any function is translated.
24b. **Trap sites** (D27, only when `opts.trap_sites`). While translating each function, give each of the following a site, numbering sites from 1 in emission order (functions in ascending `DefId`, and within a function the prologue first, then blocks in index order and instructions in list order):
   - **Prologue.** One `Entry` site per function, with `node` 0. The prologue's `StackLimit` edge calls `cprime_rt_trap_at(rt, 8, <entry site>)`. The `FuelExhausted` edge keeps `cprime_rt_trap`, since release fuel is unlimited.
   - **Trap-site instructions.** Each `Bin Add`, `Sub`, `Mul`, `Div`, `Rem` and `Index` gets a `Trap(<kind>)` site for each trap kind its check can raise, with `node = node_map[result]` (0 when absent). Every trap edge of the instruction calls `cprime_rt_trap_at(rt, code, site)`.
   - **`Intrinsic Slice`.** A `Trap(IndexOutOfBounds)` site with `node = node_map[result]`. AOT code compares `start ≤ end` and `end ≤ len(b)` inline before calling `cprime_rt_slice`, and the failing edge calls `cprime_rt_trap_at(rt, 3, site)`.
   - **`Terminator::Trap(Unreachable)`.** A `Trap(Unreachable)` site whose node follows D13: `node_map` of the first instruction of the terminator's block, else 0. The impossible-tag default of a `Switch` gets a `Trap(Unreachable)` site with node 0.
   - **`Perform`.** A `Perform { cap, op }` site with `node = node_map[result]`. Before the indirect call, AOT code stores the site id into `rt.site` (one 64-bit store at offset 56).

   With `opts.trap_sites == false`, trap edges call `cprime_rt_trap(rt, code)`, `Perform` does not store `rt.site`, `Slice` has no inline check, and `AotOutput.sites` is empty. More than `u32::MAX` sites → `CodegenError::Llvm { message: "too many trap sites" }`.
24c. **Call chain** (D28, only when `opts.call_chain`). `functions` lists every function in ascending `DefId`, and the function index is its position + 1. In each function's prologue, after the depth check, AOT code loads `rt.chain` (offset 64) and stores the function index as a `u32` at `chain[rt.depth]`. The depth check guarantees `rt.depth ≤ rt.max_depth`, so the store stays within the `max_depth + 1` entries the release main allocates. An object built with `call_chain` must run with a non-null `rt.chain`; the generated release main (`cprime-cli`) and the `par_map` worker contexts (`cprime-provider`) guarantee it. The epilogue stores nothing: entries above `rt.depth` are ignored. Without `opts.call_chain`, no prologue store is emitted.
25. **Export** `cprime_entry` as an external function with the entry function's C signature, whose body tail-calls the entry function.
25a. **Server programs** (D34, ADR-051): when `entry` is a server handler, emit every function with unwind tables (`uwtable`), declare `cprime_rt_trap` and `cprime_rt_trap_at` with the `C-unwind` ABI, and export the handler as `cprime_handler` (`extern "C-unwind" fn(*mut RtContext, u64) -> u64`) instead of `cprime_entry`. The prologue fuel check (step 4) already bounds each task, because a server task's `rt.fuel` is `limits.request_fuel`. Batch entries are emitted exactly as before.
26. **Optimise** with the LLVM default `O2` pipeline, verify the module (failure → `CodegenError::Llvm`), and write a relocatable object file to `out` (I/O failure → `CodegenError::Io`). Return `AotOutput { object: out, entry_symbol: "cprime_entry", sites, functions }`: `sites` in emission order (step 24b; empty without `trap_sites`), `functions` in ascending `DefId` (always filled). Log `info` `codegen.aot_emitted` with `object`, `functions`, `bytes`, `sites` (count), `call_chain`, `elapsed_us`.

**Part 6 — Differential testing obligations (test suite of this crate)**

27. For every corpus program and input set: the interpreter, the JIT (mode 1) and an AOT build run through a test harness with mode-1-equivalent observation (exit code mapped back to `RtTrap`) must produce equal results (`Value` equality), equal trap kinds, and — for interpreter and JIT, with `AllocTracker` attached — `live_objects == 0` after the result is dropped. A semantics test table covers every row of step 3 at its boundary values (0, 1, type maximum, type minimum, `I64::MIN / -1`, `I64::MIN % -1`, division by zero with an overflowing dividend).
28. **Trap location** (RFC 03 criterion 13). For every corpus program and input set on which the AOT build traps with a kind from `Overflow`, `DivisionByZero`, `IndexOutOfBounds`, `Unreachable` or `StackLimit`, the harness records the site id the binary reported. The build uses `AotOptions { call_chain: false, trap_sites: true }`, and the harness's test build of the provider records the `site` argument of `cprime_rt_trap_at`. It maps that id through `AotOutput.sites` to `(func, node)`. The interpreter runs the same input through `interp::replay` with an oracle that answers `Live` to every request, and must end in `Trapped { kind, func, at }` with the same kind, the same `func`, and `node_map[at]` equal to the site's node. When `at` is `None`, the site's node must be the one step 24b gives that terminator or prologue (D13 rule, or 0).
29. **Semantic preservation of RC insertion and reuse** (RFC 07 criteria 16–17, D45). For every corpus program and input set, the interpreter runs three programs built from the same lowered functions:
    - (a) **pre-RC**: the lowered functions themselves, `borrows` with every parameter owned, `RtContext` mode 1 with `mem_limit` 1,073,741,824 (1 GiB) and an `AllocTracker`, `free_all` after each run;
    - (b) **RC with reuse**: `insert_rc_opts(.., RcOptions { reuse: true })` with inferred borrows (the `insert_rc` output);
    - (c) **RC without reuse**: `insert_rc_opts(.., RcOptions { reuse: false })` with the same borrows.

    Legs (a), (b) and (c) must give equal results (`Value` equality) and equal trap kinds for `Overflow`, `DivisionByZero`, `IndexOutOfBounds`, `Unreachable` and `ProviderPanic`; legs (b) and (c) must also give `live_objects == 0` after the result is dropped. An input on which any leg ends with `FuelExhausted`, `MemoryLimit` or `StackLimit` is excluded from the comparison and counted; the test prints the excluded count per program and fails only on a difference among compared inputs. The HLD semantic-preservation table (`07-cross-cutting.md`) names this item for RC insertion and FBIP reuse.

#### Data Model

N/A — no persistent storage. In-memory structures:

| Structure | Contents | Lifetime |
|---|---|---|
| `JitModule` | Cranelift module and its executable memory; `BTreeMap<DefId, *const u8>` function addresses; operation table `Box<[Slot]>` (`Slot { f: StubFn, op_id: u32 }`) plus generated provider trampolines, with `Vec<(CapId, Symbol, OpTarget)>` initial targets | until dropped by its owner (`cprime-daemon` builds one per request and, for `assurance tested`, moves it into `cprime-verify`'s T3 provider thread, which drops it, D41) |
| Interpreter `Frame` | `def`, `block`, `inst_index`, `regs: Vec<u64>`; in `replay` also `types: Vec<Option<Type>>` indexed by `ValueId` and the frame's depth | one `run` or `replay` call |
| `Replay` | `outcome`, `events` (depth-1 activation only; one event per executed non-RC instruction, terminator or trap) | returned to the caller of `replay` |
| AOT site list | `Vec<AotSite>` and `Vec<DefId>` built during translation (steps 24b, 24c) | one `emit_object` call; returned in `AotOutput` |
| AOT LLVM context and module | per `emit_object` call | one call; only the object file persists, written by the caller's chosen path under `.cprime/` |

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Host is not a supported target (JIT or AOT) | `Err` | `CodegenError::TargetUnsupported` | `TargetUnsupported { triple: "riscv64gc-unknown-linux-gnu" }` |
| `Call` to a `DefId` absent from `functions` and `unimplemented` | `Err` | `CodegenError::MissingFunction` | `MissingFunction { def: 9f3c… }` |
| `ParMap` names a `DefId` absent from `functions` and `unimplemented` | `Err` | `CodegenError::MissingFunction` | `MissingFunction { def: 9f3c… }` |
| `Perform` without an `op_targets` entry | `Err` | `CodegenError::MissingOpTarget` | `MissingOpTarget { cap: CapId(3), op: Symbol(17) }` |
| Provider `flat_index` beyond the registry | `Err` | `CodegenError::ProviderIndexOutOfRange` | `ProviderIndexOutOfRange { flat_index: 12, available: 10 }` |
| AOT with an unbound operation | `Err` | `CodegenError::UnboundOperation` | `UnboundOperation { cap, op }` — `cprimec` reports the missing `bind` |
| AOT with an unimplemented callee | `Err` | `CodegenError::UnimplementedCallee` | `UnimplementedCallee { def }` |
| `set_op` on an operation the module never performs | `Err` | `CodegenError::NotAnOperation` | `NotAnOperation { cap, op }` |
| Cranelift rejects a function | `Err` | `CodegenError::Cranelift` | `Cranelift { def, message: "<cranelift verifier text>" }` — compiler bug, daemon failure (A-17) |
| LLVM module verification fails | `Err` | `CodegenError::Llvm` | `Llvm { message }` — compiler bug |
| Object file cannot be written | `Err` | `CodegenError::Io` | `Io { path: ".cprime/release/obj/cprime_entry.o", message: "Permission denied" }` |
| Checked arithmetic overflows at run time | trap | `RtTrap::Overflow` / exit 3 | mode 1: `rt.trap = 1`, result 0; mode 0: stderr `cprime: trap Overflow at auth/generate_token.opq:3:13 in Auth::generate_token` (location from the site table, `cprime-provider` step 8) |
| Division or remainder by zero | trap | `RtTrap::DivisionByZero` / exit 3 | `cprime: trap DivisionByZero at <file>:<line>:<column> in <spec>` |
| Index or slice out of range | trap | `RtTrap::IndexOutOfBounds` / exit 3 | `cprime: trap IndexOutOfBounds at <file>:<line>:<column> in <spec>` (the `Slice` check is inline in AOT, step 24b) |
| `Trap(Unreachable)`, stubbed operation (interpreter), unimplemented callee | trap | `RtTrap::Unreachable` / exit 3 | `cprime: trap Unreachable at <file>:<line>:<column> in <spec>` |
| `emit_object` with `call_chain: true` and `trap_sites: false` | `Err` | `CodegenError::Llvm` | `Llvm { message: "call_chain requires trap_sites" }` |
| More than `u32::MAX` trap sites in one object | `Err` | `CodegenError::Llvm` | `Llvm { message: "too many trap sites" }` |
| Oracle answers `Value(v)` whose type does not match the instruction's result type | `Replay` | `ReplayOutcome::Mismatch` | debug builds: assertion failure naming the type; release builds: `Mismatch { at }` |
| Oracle answers `Mismatch` | `Replay` | `ReplayOutcome::Mismatch` | `Mismatch { at }`; `events` hold everything recorded before `at` |
| A replay request or event needs the type of a `Call` or `Perform` result for which oracle and program both give no type | `Replay` | `ReplayOutcome::Mismatch` | `Mismatch { at }` at the instruction that needed the type; unreachable for an oracle whose `call_type` and `op_type` are complete |
| Fuel exhausted | trap | `RtTrap::FuelExhausted` | mode 1 only (release fuel is unlimited) |
| Allocation beyond `mem_limit` | trap | `RtTrap::MemoryLimit` / exit 5 | raised inside `cprime_rt_alloc` |
| Provider panics | trap | `RtTrap::ProviderPanic` / exit 6 | raised by the SDK wrapper |
| Call depth above `max_depth` | trap | `RtTrap::StackLimit` / exit 7 | mode 1: recorded in `rt.trap`; mode 0: `cprime: call depth limit exceeded in <spec> (<file>:<line>:<column>)` from the function's `Entry` site, exit 7 |
| Interpreter called with an unknown function or wrong argument count | `Err` | `RtTrap::Unreachable` | logged as `codegen.interp_bad_call` |

#### Acceptance Criteria (Gherkin)

```gherkin
Feature: cprime-codegen

  Scenario: Happy path — interpreter runs the concept example
    Given the RC-inserted program for Auth::generate_token with host::Crypto bound
    When interp::run is called with user_id 1, now 100 and a 32-byte secret
    Then it returns a SessionToken with user_id 1 and expires_at 3700
    And the sig field equals HMAC-SHA256(secret, u64_be(1) ++ u64_be(3700))

  Scenario: Happy path — JIT matches the interpreter
    Given the same program compiled with JitModule::compile
    When call is invoked with the same arguments in mode 1
    Then word_to_value of the result equals the interpreter's value
    And with an AllocTracker attached, live_objects is 0 after the result is dropped

  Scenario: Edge — I64 division and remainder boundaries are identical in all back ends
    Given a function returning (/ a b) and another returning (% a b) over I64
    When each back end runs a = -9223372036854775808 and b = -1
    Then the division traps Overflow in the interpreter, the JIT and the AOT binary (exit code 3)
    And the remainder returns 0 in all three

  Scenario: Edge — division by zero is reported before overflow
    Given a function returning (/ a b) over I64
    When a = -9223372036854775808 and b = 0
    Then every back end traps DivisionByZero

  Scenario: Edge — fuel bounds work in the JIT
    Given a Diverge-granted spec that recurses forever
    When it is called in mode 1 with fuel 1000
    Then the call returns with rt.trap = FuelExhausted after exactly 1000 successful function entries

  Scenario: Edge — fuzzing stub replaces a non-deterministic operation
    Given a JitModule whose program performs host::Clock::now_unix_seconds
    When set_op installs a stub returning 42 for that operation
    Then a call that returns the clock value returns 42
    And after reset_ops the provider's own function is called again

  Scenario: Edge — unimplemented callee traps Unreachable
    Given a program whose candidate calls a spec listed in unimplemented
    When the candidate is called through the JIT
    Then rt.trap is Unreachable and the result word is 0

  Scenario: Error path — AOT refuses an unbound operation
    Given a program whose op_targets maps CryptoCap::hmac_sha256 to Stub
    When emit_object is called
    Then it returns CodegenError::UnboundOperation for that operation

  Scenario: Error path — missing operation target
    Given a program that performs an operation absent from op_targets
    When JitModule::compile is called
    Then it returns CodegenError::MissingOpTarget

  Scenario: Happy path — AOT object exports cprime_entry
    Given a program with entry Main::main and every operation bound
    When emit_object writes to an object path
    Then the object defines the global symbol cprime_entry and references cprime_rt_op_table
    And linking it with libcprime_runtime.a and the generated main yields a binary whose output equals the interpreter's result

  Scenario: Differential — corpus agreement
    Given every corpus program and input set
    When run through the interpreter, the JIT and an AOT build
    Then all three produce equal values or equal trap kinds

  Scenario: Differential — parallel map agrees across back ends and worker counts
    Given a corpus program using par_map over 1000 elements, with one input that traps Overflow at element 600
    When it runs in the interpreter, the JIT and AOT builds with limits.workers 1 and 4
    Then every back end returns equal results for the other inputs
    And every back end reports the trap kind Overflow for the trapping input

  Scenario: Happy path — replay with a Live oracle equals run
    Given the RC-inserted program for Auth::generate_token with host::Crypto bound and an AllocTracker attached
    And an oracle that answers Live to every request
    When interp::replay is called with user_id 1, now 100 and a 32-byte secret
    Then the outcome is Returned with the value interp::run returns for the same arguments
    And events hold one Inst event for each of v3 to v9, all with answered false, followed by one Term event
    And live_objects is 0 after the result is dropped

  Scenario: Happy path — oracle Value replaces a Perform
    Given the same program and an oracle that answers Value(FixedBytes of 32 zero bytes) to perform at v8 and Live otherwise
    When interp::replay is called with user_id 1, now 100 and a 32-byte secret
    Then the provider operation hmac_sha256 is not called
    And the Inst event for v8 has value FixedBytes of 32 zero bytes and answered true
    And the outcome is Returned with sig equal to 32 zero bytes and expires_at 3700

  Scenario: Edge case — Mismatch stops the replay
    Given the same program and an oracle that answers Mismatch to perform at v8
    When interp::replay is called with user_id 1, now 100 and a 32-byte secret
    Then the outcome is Mismatch with at v8
    And the last event is the Inst event for v7
    And after AllocTracker::free_all live_objects is 0

  Scenario: Edge case — a trap inside a callee is recorded at the depth-1 call
    Given a program where Main::run calls Math::half at v5, and Math::half traps DivisionByZero at its own v2
    And an oracle that answers Live to every request
    When interp::replay runs Main::run
    Then the last event is Trap { at: Some(v5), kind: DivisionByZero }
    And the outcome is Trapped { kind: DivisionByZero, func: Math::half, at: Some(v2) }

  Scenario: Error path — AOT overflow trap reports the site of (+ now 3600)
    Given the revision-1 contract program for Auth::generate_token and AotOptions with trap_sites true and call_chain false
    When emit_object runs
    Then AotOutput.sites contains AotSite { func: Auth::generate_token, node: NodeId(6), kind: Trap(Overflow) } at site id s
    And the overflow edge of v4 calls cprime_rt_trap_at with code 1 and site s
    And the linked binary given {"user_id": 1, "now": 18446744073709551615, "secret": "0x" followed by 64 zeros} exits 3 with a stderr line starting "cprime: trap Overflow at "

  Scenario: Edge case — call_chain build writes chain entries
    Given a program where Main::run calls Auth::generate_token, built with AotOptions { call_chain: true, trap_sites: true }
    And an RtContext whose chain points to max_depth + 1 zeroed u32 entries
    When cprime_entry runs Main::run and Auth::generate_token returns
    Then chain[1] holds the function index of Main::run and chain[2] the function index of Auth::generate_token
    And the indices equal the positions + 1 of those DefIds in AotOutput.functions

  Scenario: Edge case — trap_sites false emits no sites
    Given the concept example program and AotOptions { call_chain: false, trap_sites: false }
    When emit_object runs
    Then AotOutput.sites is empty and AotOutput.functions lists Auth::generate_token
    And the object references cprime_rt_trap and does not reference cprime_rt_trap_at
    And no instruction in the object stores to offset 56 of the RtContext

  Scenario: Error path — call_chain without trap_sites is refused
    Given AotOptions { call_chain: true, trap_sites: false }
    When emit_object runs
    Then it returns CodegenError::Llvm with message "call_chain requires trap_sites" and writes no object file

  Scenario: Differential — trap locations agree between AOT and the interpreter
    Given every corpus program and input set on which the AOT build traps
    When the harness maps the reported site through AotOutput.sites and replays the input with a Live oracle
    Then the replay ends Trapped with the same kind and func, and node_map of its at equals the site's node

  Scenario: Happy path — server handler object
    Given the "api.cprime" server example with an accepted Api::handle
    When emit_object runs with entry Api::handle as a server handler
    Then the object exports cprime_handler, every function has an unwind table entry, and no symbol cprime_entry exists

  Scenario: Differential — RC insertion and reuse preserve results (RFC 07 criterion 16)
    Given the corpus with a list-map program whose RC-inserted form reuses cells
    When Part 6 item 29 runs legs pre-RC, RC with reuse and RC without reuse on every input set
    Then every compared input gives equal results and equal trap kinds in all three legs
    And legs with RC report live_objects 0 after each result is dropped
    And the test output lists the number of inputs excluded for a resource limit

  Scenario: Edge case — the interpreter runs pre-RC IR and leaks until free_all (D45)
    Given the lowered concept example without RC insertion and an AllocTracker
    When interp::run is called with user_id 1, now 100 and a 32-byte secret
    Then it returns the same value as for the RC-inserted program
    And live_objects is greater than 0 until free_all, and 0 after it
```

#### Performance, Security, Observability

- **Performance targets:** JIT compilation ≤ 12 ms p95 for one function of ≤ 200 IR instructions plus its accepted callees already in Cranelift IR form (share of T1 ≤ 20 ms, HLD G5); `JitModule::call` overhead ≤ 1 µs per call beyond the compiled code; fuzzing throughput supports 10,000 iterations within 5 s (T3) for functions of ≤ 200 instructions, i.e. ≤ 500 µs per iteration including stubs. Interpreter: no latency target (oracle); it must complete the corpus in under 60 s on CI. AOT: no latency target; release run time is measured against the 2× Rust goal on the Phase 5 benchmark suite (HLD G10). Every compiled frame ≤ 1 KiB, checked by a test on the corpus, so that `max_depth` 100,000 fits a 100 MiB stack.
- **Security:** this crate is in the trusted computing base (Cranelift, LLVM, and the code emitted here). No input arrives from outside the compiler; inputs are verified IR. Generated code contains no undefined behaviour (steps 16, 24). JIT executable memory is created by Cranelift's JIT module and freed when the `JitModule` drops. Operation-table writes require `&mut JitModule`. No secrets: secrets modelled as `Bytes` parameters are synthetic during fuzzing (HLD data lifecycle).
- **Observability:** events through the workspace logging facade (Phase 4 *Logging*): `debug` `codegen.jit_compiled` (`functions`, `insts`, `elapsed_us`); `info` `codegen.aot_emitted` (`object`, `functions`, `bytes`, `sites`, `call_chain`, `elapsed_us`); `trace` `codegen.interp_run` (`def`, `steps`, `result`); `trace` `codegen.interp_replay` (`def`, `steps`, `answered`, `outcome`); `error` `codegen.interp_bad_call` (`def`). Replay event values are never logged.
- **Release reporting cost (ADR-048, SA-60):** with `trap_sites`, the normal path pays one 64-bit store per `Perform` and the inline `Slice` range check; site ids travel only on cold trap edges. With `call_chain`, it also pays one 32-bit store per function entry. Both are measured against the `trap_sites: false` baseline by the Phase 5 benchmarks (RFC 03 criterion 16). No metrics endpoint; JIT time contributes to `timings_us.t1`.

#### Gaps

None.
