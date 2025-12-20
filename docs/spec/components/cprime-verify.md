# Component spec — cprime-verify

> **Spec set:** cprime
> **Status:** Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> **Last updated:** 2025-12-20

### SPEC: cprime-verify
**File:** `crates/cprime-verify/src/lib.rs` (modules `input.rs`, `smt/sorts.rs`, `smt/encode_ir.rs`, `smt/encode_contract.rs`, `smt/obligations.rs`, `schedule.rs`, `model.rs`, `accept.rs`, `eval.rs`, `witness.rs`, `vacuity.rs`, `concrete.rs`, `provider_thread.rs`, `fuzz/gen.rs`, `fuzz/harness.rs`, `fuzz/probe.rs`) | **Package:** `cprime-verify` | **Phase:** 4 (SMT, acceptance rules), 5 (fuzzing, `assurance tested`) | **Dependencies:** `cprime-core`, `cprime-codegen`, `cprime-provider`; third-party `z3`, `arbitrary`, `serde_json`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprime-verify` decides whether a submission that has passed T0 and T1 meets its contract. It lowers the spec's `requires`, the candidate's C′-IR and the spec's `ensures` to SMT, generates one proof obligation per trap site, call precondition, recursive call edge and `ensures` clause, solves the obligations in parallel Z3 contexts under the per-obligation timeout and the per-submission budget (T2), and, when the spec's assurance is `tested` and at least one obligation is `unknown`, runs the fuzzing harness against the JIT-compiled implementation (T3). Before any obligation it checks that the contract is not vacuous (ADR-052). It applies the acceptance rules of ADR-025 and returns one `VerificationResult`: accepted or rejected, the evidence level, the provider operations the result assumes, and diagnostics carrying counterexamples, each classified `concrete` or `abstract` (ADR-055). It writes nothing to disk and holds no state between calls; the vacuity cache lives in `cprime-daemon` (D38). Every call into provider code runs on a thread it can abandon (D41, ADR-054).

#### Approach

Each obligation is rendered as a self-contained SMT-LIB 2 script (declarations, assertions, one `check-sat`), and each worker thread parses scripts into its own Z3 context. Rendering text keeps Z3 terms out of shared memory (Z3 contexts are not shareable across threads), makes every obligation independently reproducible at `trace` level, and fixes the encoding in one place. The IR is encoded by symbolic execution over the block graph, which is a directed acyclic graph because the language has no loops (ADR-024) and lowering emits no back edges; calls to specs are modelled by the callee's contract (SA-18). Fuzzing evaluates contracts with a concrete evaluator that implements the same definedness semantics as the SMT encoding (SA-19). Rejected alternatives: building Z3 terms through the `z3` crate's term API in one shared context — it serialises all obligations onto one thread and breaks the T2 composition argument of ADR-025; bit-vector arithmetic for program integers — overflow detection is native, but contract arithmetic is mathematical (SA-19) and mixing `bv2int` across every clause makes `unknown` more frequent than range-constrained `Int`.

#### Shared Context

Copied from `phase-2-architecture.md` §2C and §2E. Only the items this crate reads or produces are reproduced.

```rust
// ---- cprime_core::ids ----
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct Symbol(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct DefId(pub [u8; 32]);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct CapId(pub u32);   // CapId(0) = Diverge
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, serde::Serialize)] #[serde(transparent)]
pub struct NodeId(pub u32);                                // pre-order index in the submitted .opq; 0 = impl form
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct ValueId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct BlockId(pub u32);

// ---- cprime_core::ir ----
pub enum Type { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<Type>), Struct(DefId), Sum(DefId),
                Fn { params: Vec<Type>, ret: Box<Type>, effects: BTreeSet<CapId> }, Token }
pub enum Const { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(Vec<u8>) }
pub enum BinOp { Add, Sub, Mul, Div, Rem, WrappingAdd, WrappingSub, WrappingMul,
                 Eq, Ne, Lt, Le, Gt, Ge, And, Or, BytesConcat }
pub enum IntrinsicOp { U64Be, Push, Slice, ToBytes }
pub enum Inst {
    Const(Const), Bin { op: BinOp, lhs: ValueId, rhs: ValueId }, Not(ValueId),
    Call { func: DefId, args: Vec<ValueId> }, Perform { cap: CapId, op: Symbol, args: Vec<ValueId> },
    Intrinsic { op: IntrinsicOp, args: Vec<ValueId> },
    MakeStruct { ty: DefId, fields: Vec<ValueId>, reuse: Option<ValueId> }, GetField { value: ValueId, index: u32 },
    MakeVariant { ty: DefId, tag: u32, fields: Vec<ValueId>, reuse: Option<ValueId> },
    GetVariantField { value: ValueId, tag: u32, index: u32 },
    MakeVec { elem: Type, items: Vec<ValueId> }, Len(ValueId), Index { vec: ValueId, index: ValueId },
    ParMap { func: DefId, xs: ValueId },
    RcDup(ValueId), RcDrop(ValueId), DropReuse(ValueId),       // never present in the SMT input (pre-RC IR)
}
pub enum TrapKind { Overflow, DivisionByZero, IndexOutOfBounds, Unreachable }
pub enum Terminator {
    Return(ValueId), Jump { target: BlockId, args: Vec<ValueId> },
    Branch { cond: ValueId, then_target: BlockId, then_args: Vec<ValueId>, else_target: BlockId, else_args: Vec<ValueId> },
    Switch { scrutinee: ValueId, cases: Vec<(BlockId, Vec<ValueId>)> }, Trap(TrapKind),
}
pub struct Block { pub params: Vec<(ValueId, Type)>, pub insts: Vec<(ValueId, Inst)>, pub term: Terminator }
pub struct Function {
    pub def: DefId, pub ty: Type, pub entry: BlockId, pub blocks: Vec<Block>, pub value_count: u32,
    pub node_map: BTreeMap<ValueId, NodeId>,   // (§2C, D3, D23) source node of every parameter, non-constant instruction, join parameter and Return value
}
pub struct Layouts { pub structs: BTreeMap<DefId, Vec<Type>>, pub sums: BTreeMap<DefId, Vec<Vec<Type>>> }

// ---- cprime_core::tast ----
pub struct TSpec { pub def: DefId, pub name: String, pub params: Vec<(Symbol, Type)>, pub ret: Type,
    pub uses: BTreeSet<CapId>, pub assurance: Assurance, pub waivers: Vec<Waiver>,
    pub requires: Vec<TClause>, pub ensures: Vec<TClause>, pub is_entry: bool }
pub struct TImpl { pub spec: DefId, pub params: Vec<(Symbol, NodeId)>, pub decreases: Option<TDecreases>,
    pub body: TExpr, pub node_count: u32 }
pub struct TDecreases { pub node: NodeId, pub measure: Vec<TExpr> }   // each element: type U64
pub struct TExpr { pub node: NodeId, pub ty: Type, pub kind: TExprKind }   // TExprKind as §2C
pub struct TClause { pub text: String, pub span: Span, pub expr: CExpr }
pub struct CExpr { pub ty: Type, pub kind: CExprKind }
pub enum CExprKind {
    Int(i128), Const(Const), Param(u32), Result,
    Bin { op: CBinOp, lhs: Box<CExpr>, rhs: Box<CExpr> }, Not(Box<CExpr>),
    And(Box<CExpr>, Box<CExpr>), Or(Box<CExpr>, Box<CExpr>), Implies(Box<CExpr>, Box<CExpr>),
    Is { value: Box<CExpr>, tag: u32 }, Field { value: Box<CExpr>, index: u32 },
    VariantField { value: Box<CExpr>, tag: u32, index: u32 },
    Intrinsic { op: IntrinsicOp, args: Vec<CExpr> }, Len(Box<CExpr>), Index { vec: Box<CExpr>, index: Box<CExpr> },
    OpApp { cap: CapId, op: Symbol, qualified: String, args: Vec<CExpr> },   // deterministic ops only
}
pub enum CBinOp { Add, Sub, Mul, Div, Rem, Eq, Ne, Lt, Le, Gt, Ge, Concat }
pub struct TerminationInfo { pub component: BTreeSet<DefId>, pub diverges: bool, pub edges: Vec<CallEdge>,
    pub diagnostics: Vec<Diagnostic> }
pub struct CallEdge { pub caller: DefId, pub callee: DefId, pub node: NodeId, pub caller_is_candidate: bool }

// ---- cprime_core::diag ----
pub enum DiagnosticKind { ParseError, UnknownName, ArityMismatch, TypeMismatch, UnauthorizedEffect,
    MissingDecreases, NonExhaustiveMatch, UnconstrainedOutput, InvalidParMap, ContractViolation, ReachableTrap,
    NonDecreasing, VerificationInconclusive, NonDeterministicProvider, InvalidServer,
    VacuousContract, VacuityUnknown }                      // (D37)
pub enum Tier { T0, T1, T2, T3 }            // serialised lower-case
pub enum Severity { Error, Note }           // serialised lower-case; Note only on VacuityUnknown (D37)
pub struct Span { pub file: String, pub line: u32, pub column: u32 }
pub struct Diagnostic { pub kind: DiagnosticKind, pub tier: Tier, pub severity: Severity, pub node: NodeId,
    pub span: Option<Span>, pub message: String, pub clause: Option<String>,
    pub counterexample: Option<Counterexample>, pub reason: Option<String>,
    pub witness: Option<WitnessTrace>,                     // (D22) serialised only when Some
    pub concrete: Option<ConcreteCheck> }                  // (D43) serialised only when Some
pub struct Counterexample { pub inputs: Vec<(String, Value)>, pub expected: Vec<(String, Value)>, pub actual: Vec<(String, Value)> }
// reason (closed set): "solver_unknown", "timeout", "budget_cut_off",
//                      "fuzz_budget_incomplete", "fuzz_resource_limit", "callee_unimplemented",
//                      "unsupported_construct" (D44), "provider_failure" (D42), "provider_timeout" (D41)

// (D43, ADR-055) concreteness of a counterexample
pub struct ConcreteCheck { pub status: ConcreteStatus, pub reason: Option<String>, pub witness: Option<WitnessTrace> }
pub enum ConcreteStatus { Concrete, Abstract }            // "concrete", "abstract"
// ConcreteCheck.reason (closed set; Some iff Abstract): "no_violation", "different_outcome", "callee_unimplemented",
//   "stubbed_operation", "resource_limit", "provider_failure", "provider_timeout"

// (D22) Witness trace types; enums serialised lower-case snake case.
pub struct WitnessTrace { pub status: WitnessStatus, pub steps: Vec<TraceStep>, pub omitted: u32 }
pub enum WitnessStatus { Reproduced, TrapFirst, NotReproduced }   // "reproduced", "trap_first", "not_reproduced"
pub struct TraceStep { pub node: NodeId, pub span: Option<Span>, pub name: Option<String>, pub kind: StepKind,
    pub value: Value, pub operands: Option<Vec<Value>>, pub source: StepSource }
pub enum StepKind { Param, Compute, Construct, Call, Perform, ParMap, Branch, Match, Return, Trap }
pub enum StepSource { Input, Computed, Solver, Generated, Provider }
// value per kind: Branch → Bool (the condition); Match → Str(<variant name>); Trap → Str(<TrapKind>);
// every other kind → the step's value. operands is Some only for Compute and Trap (operand values in order).
// span and name are left None by this crate; cprime-daemon fills them (D26).

// ---- cprime_core::value ----
pub enum Value { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(Vec<u8>), FixedBytes(Vec<u8>),
    Vec(Vec<Value>), Struct { fields: Vec<(String, Value)> },
    Variant { variant: String, fields: Vec<(String, Value)> }, Str(String) }
pub struct TypeSchema { pub structs: BTreeMap<DefId, Vec<(String, Type)>>,
                        pub sums: BTreeMap<DefId, Vec<(String, Vec<(String, Type)>)>> }

// ---- cprime_core::evidence ----
pub enum Evidence { Proved, Tested, None }
pub enum Assurance { Proved, Tested }
pub struct VerificationResult { pub accepted: bool, pub evidence: Evidence, pub assumed: Vec<String>,
    pub diagnostics: Vec<Diagnostic>, pub t2_us: u64, pub t3_us: Option<u64>,
    pub witness_us: Option<u64>,                           // (D25, D43) Some iff at least one witness or concrete replay ran
    pub vacuity: Option<VacuityOutcome> }                  // (D37) Some iff the vacuity check ran in this call
pub enum VacuityOutcome { Satisfiable, Vacuous { diagnostic: Diagnostic }, Unknown { reason: &'static str } }   // (D37, D38)

// ---- cprime_core::config ----
pub struct VerifyConfig { pub obligation_timeout_ms: u64, pub submission_budget_ms: u64, pub solver_threads: Option<u32>,
                          pub witness_max_steps: u32,      // (D29) 0 … 1024, default 64; 0 = no witness traces
                          pub provider_wait_ms: u64 }      // (D41) 10 … 600,000, default 2000
pub struct FuzzConfig { pub iterations: u64, pub time_budget_ms: u64, pub call_fuel: u64,
                        pub call_memory_bytes: u64, pub seed: Option<u64> }

// ---- cprime_core::db / program ----
pub struct AcceptedImpl { pub file: String, pub header: AcceptanceHeader, pub timpl: TImpl, pub function: ir::Function }
pub struct Program { pub functions: BTreeMap<DefId, ir::Function>, pub layouts: Layouts, pub schema: TypeSchema,
    pub borrows: BTreeMap<DefId, BorrowSig>, pub op_targets: BTreeMap<(CapId, Symbol), OpTarget>,
    pub unimplemented: BTreeSet<DefId> }
pub enum OpTarget { Provider { flat_index: u32 }, Stub }

// ---- cprime_types (via VerifyInput) ----
pub struct CapOpSig { pub name: Symbol, pub params: Vec<Type>, pub ret: Type, pub deterministic: bool }

// ---- cprime_provider ----
pub type OpFn = unsafe extern "C" fn(args: *const u64, rt: *mut RtContext) -> u64;
pub struct OpDecl { pub name: &'static str, pub params: Vec<BoundaryType>, pub ret: BoundaryType,
                    pub deterministic: bool, pub func: OpFn,
                    pub vectors: &'static [KnownAnswer] }   // (D40) not read by this crate
// (D40, ADR-054) deterministic: equal results for equal arguments, no observable effect, returns without waiting on anything outside the call
pub struct ProviderDecl { pub path: &'static str, pub crate_name: &'static str, pub crate_version: &'static str, pub ops: Vec<OpDecl>, pub thread_safe: bool, pub transport: Option<TransportDecl> }
pub struct ProviderRegistry { pub providers: Vec<ProviderDecl> }   // flat_ops(): global op index order
#[repr(C)] pub struct RtContext { pub trap: u32, pub mode: u32, pub fuel: u64, pub depth: u64, pub max_depth: u64,
                                  pub mem_limit: u64, pub mem_live: u64, pub tracker: *mut AllocTracker,
                                  pub site: u64, pub chain: *mut u32 }  // field order is ABI; site (offset 56) and chain (offset 64) are D27/D28, always 0 / null here
#[repr(u32)] pub enum RtTrap { None = 0, Overflow = 1, DivisionByZero = 2, IndexOutOfBounds = 3, Unreachable = 4,
                               FuelExhausted = 5, MemoryLimit = 6, ProviderPanic = 7, StackLimit = 8 }
pub struct AllocTracker { pub live_objects: u64, pub live_bytes: u64, pub objects: HashSet<usize> }
// AllocTracker::free_all(&mut self, rt) releases every tracked object.
// RtContext::sandbox(fuel: u64, max_depth: u64, mem_limit: u64, tracker: *mut AllocTracker) -> RtContext   (mode 1, depth 0, site 0, chain null)

// ---- cprime_codegen (called) ----
// unsafe fn JitModule::call(&self, func: DefId, args: &[u64], rt: &mut RtContext) -> u64
// fn marshal::value_to_word(v: &Value, ty: &Type, program: &Program, rt: &mut RtContext) -> u64
// fn marshal::word_to_value(w: u64, ty: &Type, program: &Program) -> Value
// fn JitModule::ops(&self) -> Vec<(CapId, Symbol, OpTarget)>                       (§2E, D4)
// fn JitModule::set_op(&self, cap: CapId, op: Symbol, f: StubFn, op_id: u32) -> Result<(), CodegenError>
// fn JitModule::reset_ops(&self)
//   JitModule is Send and !Sync (D41): it is moved into the T3 provider thread, never shared.
//   pub type StubFn = unsafe extern "C" fn(op_id: u32, args: *const u64, rt: *mut RtContext) -> u64;
//   Every Perform calls its slot as f(op_id, args, rt); op_id is the value passed to set_op, so one
//   stub function serves every stubbed operation and looks up per-operation state by op_id.
// fn interp::replay(program: &Program, func: DefId, args: &[Value], rt: &mut RtContext,
//                   registry: &ProviderRegistry, oracle: &mut dyn ReplayOracle) -> Replay        (§2E, D24)
pub trait ReplayOracle {
    fn call(&mut self, at: ValueId, depth: u64, callee: DefId, args: &[Value]) -> Answer;
    fn perform(&mut self, at: ValueId, depth: u64, cap: CapId, op: Symbol, args: &[Value]) -> Answer;
    fn par_map(&mut self, at: ValueId, depth: u64, func: DefId, xs: &Value) -> Answer;
    fn call_type(&self, callee: DefId) -> Option<Type>;              // result type of a called spec; None = use program
    fn op_type(&self, cap: CapId, op: Symbol) -> Option<Type>;        // result type of an operation; None = use program
}
pub enum Answer { Value(Value), Live, Mismatch }   // Value: use it as the result, do not execute; Live: execute as run does; Mismatch: stop
pub struct Replay { pub outcome: ReplayOutcome, pub events: Vec<ReplayEvent> }
pub enum ReplayOutcome { Returned(Value), Trapped { kind: RtTrap, func: DefId, at: Option<ValueId> }, Mismatch { at: ValueId } }
pub enum ReplayEvent {                              // recorded for depth 1 (the called function's own activation) only
    Inst { at: ValueId, value: Value, answered: bool },   // every instruction except RcDup, RcDrop, DropReuse
    Term { block: BlockId, taken: u32 },                   // Jump 0; Branch 0 = then, 1 = else; Switch = tag
    Trap { at: Option<ValueId>, kind: RtTrap },
}
```

Runtime protocols relied on (§2E): every trap-site instruction is `Bin Add|Sub|Mul|Div|Rem`, `Index`, `Intrinsic Slice`, plus `Terminator::Trap(Unreachable)`; in mode 1 a trap records `rt.trap` and returns; fuel is decremented per function entry.

Configuration keys read (via `VerifyInput`): `verify.obligation_timeout_ms` (default 250), `verify.submission_budget_ms` (default 2000), `verify.solver_threads` (default min(4, available parallelism)), `fuzz.iterations` (10000), `fuzz.time_budget_ms` (5000), `fuzz.call_fuel` (10,000,000), `fuzz.call_memory_bytes` (67,108,864), `fuzz.seed` (first 8 bytes, little-endian, of `impl_def`), `verify.witness_max_steps` (64; 0 … 1024; 0 = no witness traces, D29), `verify.provider_wait_ms` (2000; 10 … 600,000, D41).

**Parallel map (RFC 02, D17).** `Inst::ParMap { func: S, xs }` is encoded as a fresh `r : (Seq ⟦U⟧)` with `(= (seq.len r) (seq.len xs))` and `(forall ((i Int)) (! (=> (and (<= 0 i) (< i (seq.len xs))) E) :pattern ((seq.nth r i))))`, where `E` is `S`'s `ensures` with the parameter bound to `(seq.nth xs i)` and `result` to `(seq.nth r i)`, plus the range assertion of `⟦U⟧` on `(seq.nth r i)`. The form creates no obligation (`S` has no `requires`, D15). Fuzzing (T3) runs it through the JIT in mode 1, which takes the sequential path.

**Witness traces (RFC 03, ADR-047, D22–D26).** Every `ContractViolation` or `ReachableTrap` diagnostic that carries a counterexample also carries a `witness` when `verify.witness_max_steps > 0`: the executed steps the failing clause or trap depends on, in execution order (D22). `NonDecreasing`, `VerificationInconclusive` and `NonDeterministicProvider` diagnostics never carry one. `node_map` holds a node for every entry-block parameter, every non-constant instruction lowered from source and every join-block parameter (D23), so steps can be placed in the source. A witness is built by replaying the counterexample through `interp::replay` (D24) with an oracle (D25): for a T2 counterexample the oracle answers the candidate's own `Call`, `Perform` and `ParMap` instructions at depth 1 with the model's values and every other instruction is computed, so a callee is never entered; for a T3 counterexample callees and bound `deterministic` operations run for real (`Live`) and stubbed operations are served from the stub log of the violating call. Each trace reports `reproduced`, `trap_first` or `not_reproduced` (steps 9a and 12a). This crate leaves `TraceStep.span` and `TraceStep.name` empty; `cprime-daemon` fills them from the parsed submission (D26). Witness construction never changes acceptance (ADR-025): it runs only for `Sat` obligations and T3 violations, each of which already means rejection with evidence `none` (steps 10 and 12), and nothing it computes is read by the acceptance rules.

**Soundness boundaries (RFC 07, ADR-052 to ADR-055, D37–D45).**
- **Vacuity (step 6a).** Before any obligation, V1 checks that some input satisfies `requires` and V2 that some input and result satisfy `requires` and every `ensures`. `unsat` rejects with `VacuousContract` and nothing else runs; `unknown` adds a `VacuityUnknown` note and verification proceeds. A cached outcome arrives in `input.vacuity` and skips the check (D38).
- **Concreteness (steps 9b, 12b).** Every `ContractViolation` and `ReachableTrap` with a counterexample carries `concrete`. At T2 a second replay, the *concrete replay*, runs real bound `deterministic` operations and accepted callees, takes non-deterministic results from the model (they are environment inputs), and decides whether the violation recurs. At T3 the violating attempt decides it. The model replay of step 9a is unchanged and still checks the encoder.
- **Provider threads (D41).** The T3 run, the probe, the T3 witness replay and every concrete replay run on a thread that owns what it uses. A thread not finished `verify.provider_wait_ms` after its deadline is abandoned, and the result reports `provider_timeout`.
- **`deterministic` (ADR-054).** A bound `deterministic` operation is assumed to have no observable effect, so the T3 evaluator may call it for contract `OpApp`s through a memo shared with the program (D42).

#### Public Interface

```rust
pub struct VerifyInput<'a> {
    pub impl_def: DefId,
    pub spec: &'a TSpec,
    pub timpl: &'a TImpl,
    pub function: &'a ir::Function,                       // candidate IR before RC insertion
    pub callees: &'a BTreeMap<DefId, Arc<TSpec>>,         // every spec the candidate calls, and every member of termination.component
    pub cap_ops: &'a BTreeMap<CapId, Arc<Vec<CapOpSig>>>,
    pub termination: &'a TerminationInfo,
    pub members: &'a BTreeMap<DefId, Arc<AcceptedImpl>>,
    pub program: Arc<Program>,                            // (D41) Arc so a provider thread can own it
    pub jit: Option<JitModule>,                           // (D41) Some when spec.assurance == Tested; moved into the T3 provider thread
    pub registry: Arc<ProviderRegistry>,                  // (D41)
    pub vacuity: Option<VacuityOutcome>,                  // (D38) cached Satisfiable or Vacuous for spec.def; None → run step 6a
    pub bindings: &'a BTreeMap<CapId, u32>,
    pub verify_cfg: &'a VerifyConfig,
    pub fuzz_cfg: &'a FuzzConfig,
    pub interner: &'a Interner,                           // (§2E, D5) Symbol/CapId → text
}

/// §2E entry point. Never panics. An internal failure is reported fail-closed (see Internal Logic step 13).
/// Takes the input by value because it owns the JitModule (D41).
pub fn verify(input: VerifyInput) -> VerificationResult;

/// Same as `verify`, but internal failures are returned instead of being folded into a rejection.
/// cprime-daemon calls this and maps `Err` to the JSON-RPC daemon-failure error (A-17).
pub fn try_verify(input: VerifyInput) -> Result<VerificationResult, VerifyError>;

#[derive(Debug)]   // implements std::fmt::Display with the strings in the Error Table
pub enum VerifyError {
    SolverInit { message: String },                       // Z3 context or parameter creation failed
    CyclicBlockGraph { func: DefId },                     // IR precondition broken: back edge found
    UnsupportedConstruct { func: DefId, what: &'static str },  // an Inst/CExpr shape the encoder does not cover
    ModelDecode { obligation: u32, message: String },     // a sat model could not be turned into Values
    JitMissing,                                           // assurance Tested, T3 needed, input.jit is None
    MissingCalleeSpec { callee: DefId },                  // a Call targets a spec absent from input.callees
}
```

Preconditions the caller (`cprime-daemon`) guarantees:

1. T0 and T1 passed for the candidate; `termination.diagnostics` is empty.
2. `function` is the output of `cprime_types::lower` and has passed `ir_verify` at the post-lowering stage; it contains no `RcDup`, `RcDrop` or `DropReuse`.
3. `program.functions` contains the candidate (RC-inserted) and every accepted callee transitively reachable; `program.op_targets` maps `(cap, op)` to `Provider` for every operation whose capability has a binding, and to `Stub` otherwise (D4). Before T3 this crate replaces every `Stub` slot and every bound non-`deterministic` slot with its own stubs, and every bound `deterministic` slot with the memoising `fuzz_live` wrapper (step 11, D42), so fuzzing never performs real I/O (SA-30). `program.schema` covers every struct and sum reachable from the spec's parameter and result types.
4. `callees` contains the `TSpec` of every spec the candidate calls and of every member of `termination.component`.

##### Example

Input (concept §6.3): spec `Auth::generate_token` with the default assurance (`proved`), `requires user_id > 0`, `requires now <= U64::MAX - 3600`, `requires len(secret) >= 32`, `ensures result.user_id == user_id`, `ensures result.expires_at == now + 3600` and `ensures result.sig == CryptoCap::hmac_sha256(secret, u64_be(user_id) ++ u64_be(result.expires_at))`; candidate body uses `(+ now 1000)`.

Obligations generated (schedule order): `trap#0` (`Add` at node 6 may overflow), `ensures#0`, `ensures#1`, `ensures#2`. Outcomes: `trap#0` unsat (`now + 1000 ≤ U64::MAX − 2600`), `ensures#0` unsat, `ensures#1` sat, `ensures#2` unsat (both sides apply the same uninterpreted `hmac_sha256` to equal arguments).

`VerificationResult` serialised by `cprime-daemon` into the reply:

```json
{
  "accepted": false,
  "evidence": "none",
  "assumed": [],
  "t2_us": 41000,
  "witness_us": 900,
  "diagnostics": [
    {
      "kind": "ContractViolation",
      "tier": "t2",
      "severity": "error",
      "node": 0,
      "span": { "file": "auth.cprime", "line": 22, "column": 5 },
      "message": "ensures clause violated",
      "clause": "ensures: result.expires_at == now + 3600",
      "counterexample": {
        "inputs": { "user_id": 1, "now": 100, "secret": "0x0000000000000000000000000000000000000000000000000000000000000000" },
        "expected": { "result.expires_at": 3700 },
        "actual": { "result.expires_at": 1100 }
      },
      "witness": {
        "status": "reproduced",
        "omitted": 0,
        "steps": [
          { "node": 2, "span": { "file": "<submission>", "line": 1, "column": 33 }, "name": "now",
            "kind": "param", "value": 100, "source": "input" },
          { "node": 6, "span": { "file": "<submission>", "line": 2, "column": 13 }, "name": "exp",
            "kind": "compute", "value": 1100, "operands": [100, 1000], "source": "computed" },
          { "node": 19, "span": { "file": "<submission>", "line": 5, "column": 5 },
            "kind": "return",
            "value": { "user_id": 1, "expires_at": 1100,
                       "sig": "0x0000000000000000000000000000000000000000000000000000000000000000" },
            "source": "computed" }
        ]
      },
      "concrete": { "status": "concrete" }
    }
  ]
}
```

The witness comes from step 9a for `ensures#1`. The clause reads `result` only through `result.expires_at` (field 1), so the slice starts from operand 1 of the returned `MakeStruct` (`v4`, node 6) instead of from the whole struct: `v4` depends on parameter `now` (`v1`, node 2) and the constant `v3` (no step), and the `sig` computation (`v5`–`v8`, nodes 10–16) is not in the trace. The final `return` step shows the returned struct; its `sig` field is the model's value for the `hmac_sha256` result, supplied by the T2 oracle. This crate leaves `span` and `name` empty; `cprime-daemon` fills them from the submitted source (D26), shown here as the agent receives them. `concrete` comes from step 9b: the concrete replay runs the real `host::Crypto` `hmac_sha256`, returns `expires_at` 1100 again, and the clause still fails, so the counterexample is `concrete`. The vacuity check (step 6a) found the contract satisfiable and added nothing. The submitted source is the concept §4.2 text without an acceptance header, with `(+ now 1000)` on line 2.

Input values are whatever Z3's model supplies; the values above are illustrative. With `(+ now 3600)` every obligation is unsat and the result is `{"accepted": true, "evidence": "proved", "assumed": ["Auth::CryptoCap::hmac_sha256"], "diagnostics": [], ...}`.

Revision-1 contract (no `now <= U64::MAX - 3600` clause) with the corrected body: `trap#0` is sat with model `user_id = 1`, `now = 18446744073709548016`; the diagnostic is `{"kind": "ReachableTrap", "tier": "t2", "node": 6, "message": "trap Overflow reachable under requires", "counterexample": {"inputs": {"user_id": 1, "now": 18446744073709548016, "secret": "0x00…00"}, "expected": {}, "actual": {"trap": "Overflow"}}}` (node 6 is the `(+ now 3600)` form under SA-32 pre-order numbering).

#### Internal Logic

**T2 — SMT verification**

1. **Start.** Record `t2_start = Instant::now()`. Log `info` `verify.start` with fields `spec`, `impl_def` (first 8 hex characters), `assurance`. No error possible.

2. **Check the block graph.** Compute a topological order of `function.blocks` from `function.entry` with Kahn's algorithm over successor edges of every terminator. A back edge (Kahn's algorithm ends with reachable blocks whose in-degree never reached zero) yields `VerifyError::CyclicBlockGraph`. Unreachable blocks are ignored. Repeat for every member function in `input.members` that has an edge in `termination.edges`.

3. **Declare sorts** (`smt/sorts.rs`). Map each `Type` to an SMT sort:

   | C′ type | SMT sort | Range / shape axiom asserted on every term of the type entering from an opaque source* |
   |---|---|---|
   | `Unit` | `Unit` datatype with one constructor `unit` | — |
   | `Bool` | `Bool` | — |
   | `U8` | `Int` | `0 <= x <= 255` |
   | `U64` | `Int` | `0 <= x <= 18446744073709551615` |
   | `I64` | `Int` | `-9223372036854775808 <= x <= 9223372036854775807` |
   | `Bytes` | `(Seq (_ BitVec 8))` | — |
   | `FixedBytes(N)` | `(Seq (_ BitVec 8))` | `(= (seq.len x) N)` |
   | `Vec(T)` | `(Seq ⟦T⟧)` | — (element ranges asserted at `Index` results) |
   | `Struct(d)` | datatype `S_<first 16 hex of d>` with one constructor `mk_S_<…>` and selectors `S_<…>_f<i>` | field ranges asserted at `GetField` results |
   | `Sum(d)` | datatype `U_<first 16 hex of d>` with constructors `U_<…>_v<tag>` and selectors `U_<…>_v<tag>_f<i>`; testers `is-U_<…>_v<tag>` | field ranges asserted at `GetVariantField` results |

   *Opaque sources: spec parameters, the `result` of a called spec, results of `Perform` and of `OpApp`, results of `Index`, `GetField`, `GetVariantField`. Every value of an `Int` sort produced by the language is in range, so asserting the range at these points is sound and prevents spurious counterexamples. Mutually recursive datatypes are declared together in one `declare-datatypes` form in dependency (strongly connected component) order taken from `program.layouts`/`program.schema`. `Fn` and `Token` never reach the encoder; meeting either yields `VerifyError::UnsupportedConstruct`.

4. **Encode the candidate IR** (`smt/encode_ir.rs`). For each reachable block *b* in topological order declare `(declare-const reach_b Bool)`; `reach_entry` is `true`. For each incoming edge *e* from block *p* with edge condition *c_e* (`true` for `Jump`; `cond`/`(not cond)` for `Branch`; `(is-U_…_v<i> scrutinee)` for case *i* of `Switch`) assert `(= reach_b (or … (and reach_p c_e) …))` and, for each block parameter `(v, ty)` with argument *a*, `(=> (and reach_p c_e) (= v a))`. Each instruction `v = inst` declares `(declare-const v ⟦ty⟧)` and asserts `(= v ⟦inst⟧)` unconditionally (every encoding below is a total function; traps are separate obligations):

   | Instruction | Encoding |
   |---|---|
   | `Const` | literal; `Bytes` as `(seq.++ (seq.unit #xNN) …)` or `(as seq.empty (Seq (_ BitVec 8)))` |
   | `Bin Add/Sub/Mul` | `(+ a b)`, `(- a b)`, `(* a b)` |
   | `Bin Div`, `Rem` on `U8`/`U64` | `(div a b)`, `(mod a b)` (operands non-negative, so Euclidean = truncating) |
   | `Bin Div`, `Rem` on `I64` | truncation toward zero: `q = (ite (= (< a 0) (< b 0)) (div (abs a) (abs b)) (- (div (abs a) (abs b))))`, `r = (- a (* b q))`, with `abs x = (ite (< x 0) (- x) x)` |
   | `WrappingAdd/Sub/Mul` on `U8`/`U64` | `(mod (op a b) 2^w)` with *w* = 8 or 64 |
   | `WrappingAdd/Sub/Mul` on `I64` | `m = (mod (op a b) 2^64)`; `(ite (>= m 2^63) (- m 2^64) m)` |
   | `Eq`/`Ne` | `(= a b)` / `(not (= a b))` for every sort |
   | `Lt Le Gt Ge` | `<`, `<=`, `>`, `>=` on `Int` |
   | `And`/`Or` (strict) | `and`/`or` |
   | `BytesConcat` | `(seq.++ a b)` |
   | `Not` | `(not a)` |
   | `Call { func, args }` | fresh `(declare-const v ⟦ret⟧)` + range axiom; assert `(=> reach_b CalleeEnsures[params := args, result := v])` where `CalleeEnsures` is the conjunction over the callee's `ensures` of `(and def val)` (step 6). If `func` is absent from `callees` → `VerifyError::MissingCalleeSpec` |
   | `Perform` of a `deterministic` operation | `(op_c<cap>_s<op> args…)`, an uninterpreted function declared once per `(cap, op)` from `cap_ops`; range axiom on the result |
   | `Perform` of any other operation | fresh constant per instruction + range axiom. Because the block graph is acyclic, each instruction executes at most once per activation, so one constant per instruction is one value per call |
   | `Intrinsic U64Be` | `(seq.++ b7 … b0)` with byte *k* = `(seq.unit ((_ int2bv 8) (mod (div a 2^(8k)) 256)))`, most significant first (exact definition, so equality of encodings implies equality of inputs) |
   | `Intrinsic Push` | `(seq.++ v (seq.unit x))` |
   | `Intrinsic Slice` | `(seq.extract b s (- e s))` |
   | `Intrinsic ToBytes` | identity (same sort) |
   | `MakeStruct` / `MakeVariant` | constructor application (`reuse` ignored) |
   | `GetField` / `GetVariantField` | selector application + range axiom |
   | `MakeVec` | `seq.++` of `seq.unit` items, or `seq.empty` |
   | `Len` | `(seq.len a)` |
   | `Index` on `Vec` | `(seq.nth v i)` + range axiom |
   | `Index` on `Bytes`/`FixedBytes` | `(bv2nat (seq.nth b i))` |
   | `ParMap { func, xs }` | fresh `r` + length axiom + quantified `ensures` axiom with pattern `(seq.nth r i)` (D17); no obligation |
   | `RcDup`/`RcDrop`/`DropReuse` | `VerifyError::UnsupportedConstruct` (precondition 2 broken) |

   `Return(x)` in block *b* asserts `(=> reach_b (= ret x))` where `ret` is declared once with the result sort and the result range axiom is **not** asserted (it is a consequence). Each trap site records its safety condition *safe_s*: `Add/Sub/Mul` checked → result within the operand type's range; `Div` → `(not (= b 0))`, and for `I64` additionally `(not (and (= a -2^63) (= b -1)))` (`Overflow`); `Rem` → `(not (= b 0))` only, because `I64::MIN % -1` = 0 and does not trap (D12; the truncating encoding of step 4 yields `r = 0` for that pair). Division by zero is classified `DivisionByZero` even when the dividend is `I64::MIN` (D12); `Index` → `(and (<= 0 i) (< i (seq.len v)))`; `Slice` → `(and (<= s e) (<= e (seq.len b)))`; `Trap(Unreachable)` → `false` at the block (its condition is `(not reach_b)`).

5. **Encode the measures and member functions** (Decreases only). Each `TDecreases.measure` element is a pure `TExpr`; it is encoded by the same table over the `TExpr` tree with locals bound by `Let`/`Match` to fresh constants. A measure containing `Call` or `Perform` cannot be compared and yields a `NonDecreasing` diagnostic (step 10) with message `measure must not call specs or perform operations`. Measure arithmetic is mathematical with definedness *0 ≤ m ≤ U64::MAX* per element. Member functions from `input.members[..].function` are encoded with the step 4 rules under a name prefix `m<k>_` (call results of specs absent from `callees` are unconstrained apart from the range axiom — weaker, never unsound).

6. **Encode contracts** (`smt/encode_contract.rs`, SA-19). Each `CExpr` encodes to a pair ⟨val, def⟩. Integers are mathematical (no range check, no overflow):

   | `CExprKind` | val | def |
   |---|---|---|
   | `Int`, `Const`, `Param(i)`, `Result` | literal / parameter constant / `ret` | `true` |
   | `Bin Add Sub Mul` | `+ - *` | `(and dl dr)` |
   | `Bin Div Rem` | truncating forms of step 4 | `(and dl dr (not (= r 0)))` |
   | `Bin Concat` | `seq.++` | `(and dl dr)` |
   | `Bin Eq Ne Lt Le Gt Ge` | as step 4 | `(and dl dr)` |
   | `Not(a)` | `(not va)` | `da` |
   | `And(a, b)` | `(and va vb)` | `(and da (or (not va) db))` |
   | `Or(a, b)` | `(or va vb)` | `(and da (or va db))` |
   | `Implies(a, b)` | `(or (not va) vb)` | `(and da (or (not va) db))` |
   | `Is { value, tag }` | tester | `d` |
   | `Field` | selector | `d` |
   | `VariantField { tag, .. }` | selector | `(and d (is-…_v<tag> v))` |
   | `Len` | `seq.len` | `d` |
   | `Index` | as step 4 | `(and d di (<= 0 i) (< i (seq.len v)))` |
   | `Intrinsic Slice` | as step 4 | `(and ds de db (<= s e) (<= e (seq.len b)))` |
   | `Intrinsic U64Be / Push / ToBytes` | as step 4 | conjunction of argument defs |
   | `OpApp` | the same uninterpreted function as step 4 | conjunction of argument defs |

   A clause *holds* iff `(and def val)`. `Requires` = conjunction of all `requires` clauses' *holds* (an undefined `requires` clause therefore excludes the input). Parameter range axioms are asserted for every spec parameter.

6a. **Check the contract for vacuity** (D37, ADR-052, SA-71). Skipped when `input.vacuity` is `Some` (the cached outcome is used: `Vacuous { diagnostic }` → go to step 10 with that diagnostic; `Satisfiable` → step 7) and when `spec.is_entry`. Otherwise build at most two scripts from the step 3 and step 6 declarations, each beginning with `(set-option :produce-unsat-cores true)` and ending with `(check-sat)`:
    - **V1** (only when `spec.requires` is non-empty): every parameter range axiom; for each `requires` clause *i*, `(assert (! <holds of clause i> :named r<i>))`.
    - **V2** (only when `spec.ensures` is non-empty and V1 is not `unsat`): the V1 assertions; `ret` declared with the result sort and its range axiom asserted; for each `ensures` clause *j*, `(assert (! <holds of clause j> :named e<j>))`.

    Run them in order on the calling thread in one Z3 context with parameter `timeout` = `obligation_timeout_ms`, interrupted at `t2_start + submission_budget_ms` by the step 8 coordinator rule. Classify each with the step 8 rules.
    - Both `sat` (or skipped) → `VacuityOutcome::Satisfiable`; continue at step 7.
    - V1 or V2 `unsat` → call `get-unsat-core`; the named clauses in the core, in clause order, give `clause` = `requires: <texts joined by " && ">`, followed by `; ensures: <texts joined by " && ">` when the core names any `ensures` clause (only the `ensures` part when no `requires` clause is in the core). Build one diagnostic `VacuousContract`, tier `t2`, severity `error`, `node` 0, `span` = span of the first clause in the core, `message` = `no input satisfies requires` (V1) or `no result satisfies ensures for any input satisfying requires` (V2). Outcome `Vacuous { diagnostic }`; no obligation is generated or solved; go to step 10.
    - Either `unknown` → `VacuityOutcome::Unknown { reason }` with the step 8 reason; record one diagnostic `VacuityUnknown`, tier `t2`, severity `note`, `node` 0, `span` = span of the first `requires` clause, else of the first `ensures` clause, `reason`, `message` = `could not decide whether the contract is satisfiable`; continue at step 7.

    Set `result.vacuity = Some(outcome)` whenever the check ran. Log `debug` `verify.vacuity` with `outcome` (`satisfiable`, `vacuous`, `unknown`), `scripts` (1 or 2) and `duration_us`. Time spent here counts in `t2_us` and against the submission budget; obligations of step 8 start only after it.

7. **Generate obligations** (`smt/obligations.rs`). Each obligation is one script: shared declarations + `Requires` + the assumptions listed + `(assert (not Goal))` + `(check-sat)`.

   | Kind | One per | Assumptions besides `Requires` | Goal | Diagnostic on `sat` |
   |---|---|---|---|---|
   | `TrapSite` | trap-site instruction and `Trap(Unreachable)` block | safety of every trap site earlier in the same block and in every block that dominates it | `(=> reach_b safe_s)` | `ReachableTrap`, `node` = `function.node_map[v]` (`Unreachable`: node of the terminator's block's first instruction, else 0) |
   | `CallRequires` | (`Call` site × callee `requires` clause) | as `TrapSite`, for the call instruction | `(=> reach_b (and def val))` of the callee clause with `params := args` | `ContractViolation`, `clause` = `requires of <Callee>: <clause text>` |
   | `Decreases` | `CallEdge` in `termination.edges` when `termination.diverges` is false or the edge's caller is not the candidate | caller's `requires` (from `callees`, or the candidate spec), path to the call site in the caller function, safety of every dominating trap site | measures defined and `callee_measure(args) <_lex caller_measure(params)`; `<_lex` over equal-length tuples is `(or (< a1 b1) (and (= a1 b1) (or (< a2 b2) …)))` | `NonDecreasing`, `node` = `edge.node` when `caller_is_candidate`, else 0 with `message` naming the caller |
   | `Ensures` | `ensures` clause | safety of **every** trap site under its reachability (traps are their own obligations), every call-site `CalleeEnsures` | *holds* of the clause | `ContractViolation`, `node` 0, `span` = clause span, `clause` = `ensures: <text>` |

   Obligations are numbered 0… in schedule order (SA-24): all `TrapSite` in topological block order then instruction order; then `CallRequires` in the same order then clause order; then `Decreases` in `termination.edges` order; then `Ensures` in clause order. Log `debug` `verify.obligations` with the count per kind.

8. **Schedule and solve** (`schedule.rs`). Thread count *n* = `verify_cfg.solver_threads` or `min(4, std::thread::available_parallelism())`. Spawn *n* scoped worker threads; each creates its own Z3 context with parameter `timeout` = `obligation_timeout_ms` (creation failure → `VerifyError::SolverInit`). Workers take obligations from a shared atomic index in schedule order, parse the script with Z3's SMT-LIB 2 parser, run `check-sat`, and record the outcome with its duration. A coordinator waits on a deadline `t2_start + submission_budget_ms`; at the deadline it sets a stop flag and calls Z3's context interrupt (C API `Z3_interrupt`) on each worker's context. Outcome classification:
   - `unsat` → `Unsat`;
   - `sat` → `Sat(model)`;
   - `unknown` after the coordinator interrupt, or never started because the stop flag was set → `Unknown("budget_cut_off")`;
   - `unknown` with Z3 reason `timeout` or `canceled` and no coordinator interrupt → `Unknown("timeout")`;
   - any other `unknown` → `Unknown("solver_unknown")`.
   A Z3 parse error on a generated script is an encoder defect → `VerifyError::UnsupportedConstruct { what: "smt-lib parse" }`. Log `debug` `verify.obligation` per obligation: `id`, `kind`, `outcome`, `duration_us`; `trace` additionally the script text. `t2_us` = elapsed since `t2_start` when the last worker joins.

9. **Extract counterexamples** (`model.rs`). For each `Sat` obligation, evaluate in the model with model completion: every spec parameter → `Value` via the sort and `program.schema` (`Int` → `U8`/`U64`/`I64` by the parameter type; byte sequences → `Bytes`/`FixedBytes`; datatypes → `Struct`/`Variant` with field and variant names from the schema); parameter names come from `input.interner`. `expected`/`actual` follow §2C: for a clause whose top `CExpr` is `Bin Eq`, the left operand's text is the clause text up to the single top-level `==` token (scan outside parentheses, ignoring `==>`), trimmed; `expected` = [(left text, value of right operand in the model)], `actual` = [(left text, value of left operand in the model)]; for any other clause `expected` = [(clause text, `Bool(true)`)], `actual` = [(clause text, `Bool(false)`)]; for `TrapSite` `expected` = [], `actual` = [("trap", `Str(<TrapKind>)`)]; for `Decreases` `expected` = [("measure", `Str("decreasing")`)], `actual` = [("callee measure", tuple as `Vec` of `U64`), ("caller measure", tuple)]. Values are model evaluations only; nothing is re-executed. A model value that does not fit its type (impossible when the axioms hold) → `VerifyError::ModelDecode`.

9a. **Build witness traces (T2)** (`witness.rs`, ADR-047, D25). Skipped entirely when `verify_cfg.witness_max_steps == 0`. Otherwise, for each `Sat` obligation in schedule order whose kind is `TrapSite`, `CallRequires` or `Ensures` (`Decreases` obligations get no witness):

    a. **Oracle values.** For every instruction `v` of `input.function` that is a `Call`, `Perform` or `ParMap`, evaluate the model constant declared for `v` in step 4 with model completion and decode it to a `Value` with the type the encoder declared for it: the callee's `TSpec.ret` from `input.callees` for `Call`, `CapOpSig.ret` from `input.cap_ops` for `Perform`, `Vec(ret)` of the mapped spec from `input.callees` for `ParMap`. For an `Ensures` obligation also decode the model constant `ret` with `spec.ret` as `model_ret`. For a `CallRequires` obligation also decode the model constant of each argument of the call instruction as `model_args`. A decode failure → `VerifyError::ModelDecode`.

    b. **Replay.** Create a fresh `AllocTracker` and `rt = RtContext { trap: 0, mode: 1, fuel: fuzz_cfg.call_fuel, depth: 0, max_depth: 50000, mem_limit: fuzz_cfg.call_memory_bytes, mem_live: 0, tracker, site: 0, chain: null_mut() }`. Call `interp::replay(input.program, input.spec.def, <ModelValues.inputs in parameter order>, &mut rt, input.registry, &mut t2_oracle)` on the calling thread (the interpreter runs on an explicit frame stack). The T2 oracle answers `call`, `perform` and `par_map` requests at depth 1 with `Answer::Value` of the oracle value decoded in 9a.a for `at`, and any request at depth > 1 with `Answer::Mismatch` (depth-1 requests are always answered, so no callee is entered). `ValueId`s of `input.function` and of `program.functions[spec.def]` agree because `cprime-memory` renumbers no existing value.

    c. **Replayed values.** The replayed value of a `ValueId` `x` is: the input for an entry-block parameter; the `value` of the `Inst { at: x }` event for an instruction; for any other block parameter, the replayed value of the argument that the last `Term` event whose taken successor is `x`'s block passed in `x`'s position.

    d. **Status.**
       - `Ensures` → `reproduced` iff the outcome is `Returned(v)` and `v == model_ret`.
       - `TrapSite { value: Some(x), trap }` → `reproduced` iff the outcome is `Trapped { kind, func, at }` with `kind` equal to `trap`, `func == spec.def` and `at == Some(x)`. `TrapSite { value: None, .. }` (`Trap(Unreachable)`) → `reproduced` iff the outcome is `Trapped { kind: Unreachable, func: spec.def, at: None }` and the block it trapped in (the successor taken by the last `Term` event, or the entry block when there is none) is the obligation's block.
       - `CallRequires { call, .. }` → `reproduced` iff an `Inst { at: call }` event exists and the replayed value of each argument of `call` equals the matching entry of `model_args`.
       - Otherwise, an outcome `Trapped` with `kind` ∈ {`Overflow`, `DivisionByZero`, `IndexOutOfBounds`, `Unreachable`} and `func == spec.def` → `trap_first` (the model passed an underspecified operation on a path the obligation did not assume safe).
       - Any other outcome (`Mismatch`, a resource trap, a return where a trap was expected, a different returned value) → `not_reproduced`; log `error` `verify.witness_mismatch` with `obligation` (id), `kind` and `outcome`.

    e. **Start set.**
       - `Ensures` with outcome `Returned`: let `r` be the value of the executed `Return` terminator. If the clause's `CExpr` mentions `Result` only as the direct operand of `Field { value: Result, index: i }` nodes and `r` is defined by a `MakeStruct` instruction with an `Inst` event, the start set is the field operands at those indices `i`; otherwise it is `{r}`.
       - `TrapSite` (reproduced) and `trap_first`: the operands of the trapping instruction (none for `Trap(Unreachable)`).
       - `CallRequires`: the arguments of `call`.
       - `not_reproduced`: the value named by `Mismatch { at }`; for a trap, the trapping instruction's operands; for a return, as `Ensures`.

    f. **Slice.** Close the start set under: (1) the operands of each instruction in the set (`Bin` lhs and rhs; `Not`; `Call`, `Perform`, `Intrinsic` arguments; `MakeStruct`, `MakeVariant` fields, not `reuse`; `GetField`, `GetVariantField` value; `MakeVec` items; `Len`; `Index` vec and index; `ParMap` xs); (2) for each block parameter in the set, the argument passed to it per 9a.c; (3) the `cond` of every executed `Branch` and the `scrutinee` of every executed `Switch` (the path conditions); repeat until nothing is added.

    g. **Steps**, in execution order:
       - Entry-block parameters in the set with a `node_map` entry: `kind: param`, `value` = the input, `source: input`.
       - Each `Inst { at, value, answered }` event with `at` in the set and a `node_map` entry, except `Const`, `RcDup`, `RcDrop`, `DropReuse`: `kind` = `compute` for `Bin`, `Not`, `Intrinsic`, `Len`, `Index`, `GetField`, `GetVariantField` (with `operands` = the replayed values of its operands in order); `construct` for `MakeStruct`, `MakeVariant`, `MakeVec`; `call`; `perform`; `par_map`; `source` = `solver` when `answered`, else `computed`.
       - A block parameter in the set with a `node_map` entry, when its block is entered: `kind: compute`, `value` = its replayed value, `operands` = [that value], `source: computed`.
       - Each `Term` event of a `Branch` or `Switch` whose condition or scrutinee has a `node_map` entry: `kind: branch`, `value` = `Bool(cond)`, or `kind: match`, `value` = `Str(<variant name of the taken tag from program.schema>)`; `node` = the condition's or scrutinee's node; `source: computed`.
       - Last, for a trap outcome in the candidate: `kind: trap`, `node` = `node_map[at]` (for `Trap(Unreachable)`, the D13 rule), `value` = `Str(<TrapKind>)`, `operands` = the replayed operand values, `source: computed`. For a `Returned` outcome under `Ensures`: `kind: return`, `node` = `node_map[r]`, `value` = the returned value, `source: computed`; when `r` already produced a step, that step's `kind` becomes `return` and no step is added.
       - `node` is the `node_map` entry; `span` and `name` stay `None` (D26).

    h. **Truncate.** When there are more than `witness_max_steps` steps, keep the last `witness_max_steps` and set `omitted` to the number dropped; otherwise `omitted = 0`.

    i. **Attach.** Set the obligation's diagnostic `witness = Some(WitnessTrace { status, steps, omitted })`. Call `tracker.free_all(&mut rt)`. Add the elapsed time of 9a.a–9a.i to `witness_us`. Log `debug` `verify.witness` with `obligation`, `status`, `steps` (count), `omitted`, `duration_us`.

9b. **Classify concreteness (T2)** (`concrete.rs`, D43, ADR-055, SA-78). Runs for each `Sat` obligation of kind `TrapSite`, `CallRequires` or `Ensures`, in schedule order, whatever `verify_cfg.witness_max_steps` is (`Decreases` obligations get no `concrete`). The oracle values and `model_ret`/`model_args` are those of step 9a.a, decoded here when step 9a was skipped.

    a. **Thread.** All concrete replays of one `verify` call run in schedule order on one provider thread (D41) that owns clones of `input.program` and `input.registry` and the decoded values. The caller waits for each replay's result until `provider_wait_ms` after that replay started. On expiry the thread is abandoned (D41) and this obligation and every later one in this call get `ConcreteCheck { status: Abstract, reason: "provider_timeout", witness: None }`. When the process already has 8 abandoned threads, no thread is spawned and every obligation gets the same.

    b. **Concrete oracle** (`ConcreteOracle`): `call` and `par_map` at any depth → `Answer::Live` when the callee (or mapped spec) is in `program.functions`; when it is in `program.unimplemented` → record stop reason `callee_unimplemented` and answer `Mismatch`. `perform` at any depth of an operation whose `op_targets` entry is `Provider` and whose `CapOpSig` is `deterministic` → `Live`; of a `deterministic` operation with target `Stub` → record `stubbed_operation`, `Mismatch`; of a non-`deterministic` operation at depth 1 → `Answer::Value(<oracle value of step 9a.a for at>)`; of a non-`deterministic` operation at depth > 1 → `Answer::Value(<value generated by the step 11a generator for the operation's result type, from a SplitMix64 stream seeded with the obligation id and a per-request counter>)` (a non-deterministic result is an environment input, so any value of its type is a possible run). `call_type`/`op_type` as `T2Oracle`.

    c. **Replay.** Fresh `AllocTracker`; `rt = RtContext::sandbox(fuzz_cfg.call_fuel, 50000, fuzz_cfg.call_memory_bytes, tracker)`; `interp::replay(&program, spec.def, <inputs>, &mut rt, &registry, &mut concrete_oracle)`.

    d. **Status.** In this order:
       - a stop reason was recorded → `Abstract` with that reason;
       - outcome `Trapped` with kind `FuelExhausted`, `MemoryLimit` or `StackLimit` → `Abstract`, `resource_limit`; kind `ProviderPanic` → `Abstract`, `provider_failure`;
       - `Ensures`: outcome `Returned(v)` → evaluate the clause with the step 11b evaluator on the inputs and `v`, applying `OpApp`s through the bound providers as in D42 (a `deterministic` `OpApp` with no binding → `Abstract`, `stubbed_operation`); the clause does not hold, or is undefined → `Concrete`; it holds → `Abstract`, `no_violation`. Outcome `Trapped` with an ordinary kind → `Abstract`, `different_outcome`;
       - `TrapSite`: the outcome satisfies the step 9a.d `reproduced` rule for this obligation → `Concrete`; `Returned` → `Abstract`, `no_violation`; any other ordinary trap → `Abstract`, `different_outcome`;
       - `CallRequires { call, callee, clause }`: when the replay reached `call` (an `Inst { at: call }` event or a depth-1 `Trap { at: Some(call) }` event exists), evaluate the callee's `requires` clause `clause` (from `input.callees`) with the step 11b evaluator on the replayed arguments (step 9a.c); it does not hold, or is undefined → `Concrete`; it holds → `Abstract`, `no_violation`. When `call` was not reached: `Returned` → `Abstract`, `no_violation`; an ordinary trap → `Abstract`, `different_outcome`.

    e. **Witness.** Only for `Abstract` with reason `no_violation` or `different_outcome`, and `witness_max_steps > 0`: slice and build steps from this replay's events exactly as steps 9a.e–9a.h (start set by the outcome actually reached; `Ensures` and `CallRequires` use the 9a.e rules of their kind; status of the trace = `reproduced` when the outcome is the one this step classified, which it always is). Sources: entry parameters `input`; a `perform` answered `Live` → `provider`; a `perform` answered from the model → `solver`; a generated answer → `generated`; every other step → `computed`.

    f. **Attach.** Set the obligation's diagnostic `concrete = Some(ConcreteCheck { status, reason, witness })`. Call `tracker.free_all(&mut rt)`. Add the elapsed time of 9b.b–9b.f to `witness_us`. Log `debug` `verify.concrete` with `obligation`, `status`, `reason`, `duration_us`. Concreteness never changes acceptance (ADR-055): every obligation it runs for already rejects.

10. **Apply T2 acceptance** (`accept.rs`, ADR-025):

    | Assurance | T2 outcomes | T3 | Result | Evidence | Diagnostics |
    |---|---|---|---|---|---|
    | any | vacuity `Vacuous` (step 6a) | not run | rejected | `none` | the one `VacuousContract` diagnostic |
    | any | ≥ 1 `Sat` | not run | rejected | `none` | one per `Sat` obligation, schedule order; `Unknown` obligations add nothing |
    | any | all `Unsat` | not run | accepted | `proved` | none |
    | `proved` | no `Sat`, ≥ 1 `Unknown` | not run | rejected | `none` | `VerificationInconclusive`, tier `t2`, one per `Unknown` obligation, `reason` = its reason, `message` = `obligation <kind> #<id> not decided` |
    | `tested` | no `Sat`, ≥ 1 `Unknown` | runs | see T3 table (step 12) | | |

    Measures rejected in step 5 count as `Sat` with that `NonDecreasing` diagnostic (no counterexample). A `VacuityUnknown` note (step 6a) is appended after every other diagnostic in every row, accepted or rejected; notes never change a row (ADR-052).

**T3 — fuzzing (only for row 4)**

11. **Prepare.** If `input.jit` is `None` → `VerifyError::JitMissing`. If `program.unimplemented` is non-empty → no iteration runs; go to step 12 with outcome `Incomplete("callee_unimplemented")`. Record `t3_start`; deadline = `t3_start + fuzz_cfg.time_budget_ms`. Seed = `fuzz_cfg.seed` or `u64::from_le_bytes(impl_def.0[0..8])`. Install stubs (D4): call `JitModule::ops()`; for every `(cap, op)` whose target is `OpTarget::Stub`, and for every `(cap, op)` whose target is `Provider` but whose operation is not `deterministic`, assign a dense `op_id` (position in this list) and call `JitModule::set_op(cap, op, fuzz_stub, op_id)`; for every `(cap, op)` whose target is `Provider` and whose operation is `deterministic`, assign the next `op_id` and call `JitModule::set_op(cap, op, fuzz_live, op_id)` (D42). `fuzz_live` stores `op_id` in the thread's `in_provider` slot (0 = none; `op_id + 1` otherwise), looks the arguments up in the shared memo `FuzzState.live_memo`, on a miss calls the provider's `OpFn` with the caller's `rt` and stores the result `Value` (unless `rt.trap` is set), clears `in_provider`, and returns the word. `fuzz_stub` is one `StubFn` that indexes the per-operation stub state by `op_id`. The stub for a `deterministic` operation is memoising (a `HashMap<(CapId, Symbol, Vec<Value>), Value>` per T3 run, generating a fresh value on a miss), the stub for any other operation returns a fresh generated value per call (SA-30). Every stub call, memoised or fresh, also appends `(cap, op, <returned Value>)` to `FuzzState.stub_log`, which is cleared at the start of each attempt, so after a violation it holds the stubbed results of the violating call in call order (D25). Stub state lives in a thread-local set before each call and cleared after. After the T3 loop and the probe, call `JitModule::reset_ops()` on every exit path except abandonment (D41: an abandoned thread never resets, so its stubs stay installed).

    **Provider thread (D41).** Steps 11a–11e, the determinism probe and step 12a run on the T3 provider thread, spawned with `std::thread::Builder::new().stack_size(67108864)` (not scoped), which takes ownership of `input.jit`, clones of `input.program` and `input.registry`, the spec's clauses and the configuration, and returns `FuzzOutcome` and the violation record through a channel. When the process already has 8 abandoned threads, no thread is spawned and step 12 reports `Incomplete("provider_timeout")` with message `8 abandoned provider calls are still running; restart cprimed`. The calling thread waits until `t3_start + fuzz_cfg.time_budget_ms + verify_cfg.provider_wait_ms`; on expiry it reads `in_provider` to name the operation in progress, sets `abandoned`, increments the process-wide abandoned count, logs `warn` `verify.provider_abandoned`, drops the `JoinHandle` and goes to step 12 with `Incomplete("provider_timeout")`. The thread checks `abandoned` before each attempt and before the probe; when set it frees its tracker, decrements the count, drops the `JitModule` and exits without sending.

    Then loop until `completed == fuzz_cfg.iterations` or the deadline passes:

    a. **Generate** (`fuzz/gen.rs`). Fill a 4,096-byte buffer from a SplitMix64 stream seeded with `seed` and the attempt counter, wrap it in `arbitrary::Unstructured`, and build one `Value` per spec parameter: `Bool`/`U8`/`U64`/`I64` from `arbitrary`; `Bytes` length 0…256; `FixedBytes(N)` exactly *N* bytes; `Vec` length 0…16; sums choose a tag uniformly; nesting depth ≤ 8, beyond which the generator picks the first field-free variant if one exists, else empty vectors, else discards the attempt. One value in four for integer parameters is drawn from the boundary set {0, 1, MAX−1, MAX} (and MIN, −1 for `I64`).

    b. **Filter** (`eval.rs`). Evaluate `Requires` with the concrete evaluator. The evaluator implements the step 6 table on `Value`s with integers as `i128`: an `i128` overflow makes the expression undefined; undefined `requires` rejects the input; `OpApp` of an unbound operation reads the memo table shared with the JIT stubs; `OpApp` of a bound `deterministic` operation reads `FuzzState.live_memo`, shared with `fuzz_live`, and on a miss calls the provider's `OpFn` with a fresh `RtContext::sandbox(call_fuel, 50000, call_memory_bytes, <own tracker>)`, setting `in_provider` around the call (D42). After that call, `rt.trap == ProviderPanic` ends the loop with `Incomplete("provider_failure")` naming the operation, and the result word is not decoded; any other recorded trap is a resource discard; otherwise the word is decoded with `word_to_value`, stored in `live_memo`, and the tracker freed. Contract and program therefore see the same function, and a contract calls the provider only for argument tuples the program did not already use (SA-30). Any use of an unbound `deterministic` operation's value, by a stub or by the evaluator, sets `FuzzState.used_unbound_deterministic` for the attempt. Rejected attempts are not counted.

    c. **Run** (`fuzz/harness.rs`). Fresh `RtContext { trap: 0, mode: 1, fuel: call_fuel, depth: 0, max_depth: 50000, mem_limit: call_memory_bytes, mem_live: 0, tracker, site: 0, chain: null_mut() }`; marshal arguments with `marshal::value_to_word`; `unsafe JitModule::call(impl spec DefId, args, rt)`. Every T3 call runs on the T3 provider thread (D41), whose 64 MiB stack covers `max_depth` 50,000 × 1 KiB frames plus 1 MiB (D6).

    d. **Judge.** `rt.trap` ∈ {`Overflow`, `DivisionByZero`, `IndexOutOfBounds`, `Unreachable`} → violation `ReachableTrap` (tier `t3`, `node` 0, counterexample inputs = generated values, `actual` = [("trap", `Str(kind)`)]). `rt.trap` ∈ {`FuelExhausted`, `MemoryLimit`, `ProviderPanic`, `StackLimit`} → discard; increment `discarded_resource`. Otherwise `word_to_value(result)` and evaluate each `ensures`; the first clause that does not hold (or is undefined) → violation `ContractViolation` (tier `t3`, `clause` = `ensures: <text>`, expected/actual as step 9 with concrete values); an `i128` overflow during `ensures` evaluation → discard as resource. No violation → `completed += 1`. On a violation, record the attempt's generated inputs as `FuzzState.violating_inputs`, `rt.trap` as `violating_trap`, and for `ContractViolation` the returned value `word_to_value(result)` as `violating_result` and the violated clause index as `violating_clause`, before step e frees the heap.

    e. **Clean up.** `tracker.free_all(rt)` after every attempt that called the JIT, whatever the outcome. A violation ends the loop.

    After the loop, if no violation and `completed == iterations`, run the **determinism probe** (`fuzz/probe.rs`): for each `deterministic` operation with `OpTarget::Provider` performed by the candidate, 32 times: generate arguments, call the provider `OpFn` twice with the same marshalled arguments and a fresh `RtContext` (mode 1), compare results with `word_to_value`; a difference → `NonDeterministicProvider` (tier `t3`, `node` = `node_map` of the first `Perform` of that operation, `message` = `deterministic operation <qualified> returned different results for identical arguments`). The probe respects the T3 deadline; passing the deadline before it finishes → `Incomplete`.

12. **Apply T3 acceptance:**

    | T3 outcome | Result | Evidence | Diagnostics |
    |---|---|---|---|
    | violation found | rejected | `none` | the one violation diagnostic |
    | `NonDeterministicProvider` | rejected | `none` | that diagnostic |
    | `completed == iterations` and probe passed | accepted | `tested` | none |
    | `Incomplete` (deadline, `discarded_resource > 0`) | rejected | `none` | `VerificationInconclusive`, tier `t3`, `reason` `fuzz_resource_limit` |
    | `Incomplete` (deadline, no resource discards) | rejected | `none` | `VerificationInconclusive`, tier `t3`, `reason` `fuzz_budget_incomplete` |
    | `Incomplete("callee_unimplemented")` | rejected | `none` | `VerificationInconclusive`, tier `t3`, `reason` `callee_unimplemented` |
    | `Incomplete("provider_failure")` (D42) | rejected | `none` | `VerificationInconclusive`, tier `t3`, `reason` `provider_failure`, `message` `provider operation <q> panicked while a contract was evaluated` |
    | `Incomplete("provider_timeout")` (D41) | rejected | `none` | `VerificationInconclusive`, tier `t3`, `reason` `provider_timeout`, `message` `provider operation <q> did not return within verify.provider_wait_ms` (`<q>` from `in_provider`; `fuzzing did not finish within verify.provider_wait_ms` when no operation was in progress), or the abandoned-limit message |

    `t3_us` = elapsed since `t3_start`. Log `info` `verify.fuzz` with `completed`, `rejected_by_requires`, `discarded_resource`, `outcome`.

12a. **Build the witness trace (T3)** (`witness.rs`, ADR-047, D25). Runs only when step 12 reports a violation (`ReachableTrap` or `ContractViolation` of tier `t3`) and `verify_cfg.witness_max_steps > 0`; not counted in `t3_us`.

    a. **Replay.** Create a fresh `AllocTracker` and the `RtContext` of step 11c. Call `interp::replay(input.program, input.spec.def, FuzzState.violating_inputs, &mut rt, input.registry, &mut t3_oracle)` on the fuzz thread. The T3 oracle answers: `call` and `par_map` → `Answer::Live` at every depth (callees run for real, as they did in the JIT); `perform` of an operation whose `op_targets` entry is `Provider` and whose operation is `deterministic` → `Answer::Live` (the bound provider is called, as the JIT called it); any other `perform`, at any depth → the next entry of `FuzzState.stub_log`: `Answer::Value(value)` when the entry's `(cap, op)` equals the request's, `Answer::Mismatch` when it differs or the log is exhausted.

    b. **Status.** `ReachableTrap` → `reproduced` iff the outcome is `Trapped { kind, .. }` with `kind == violating_trap`. `ContractViolation` → `reproduced` iff the outcome is `Returned(v)` and `v == violating_result`. Any other outcome → `not_reproduced` (T3 inputs and stubbed values are concrete, so a difference means the interpreter and the JIT disagree); log `error` `verify.witness_mismatch` with `obligation` = `"t3"`, `kind` and `outcome`.

    c. **Start set and slice.** `ContractViolation`: the step 9a.e `Ensures` rule applied to `ensures[violating_clause]`. `ReachableTrap` trapping in the candidate (`func == spec.def`): the trapping instruction's operands. `ReachableTrap` trapping in a callee: the arguments of the `Call` or `ParMap` in progress, which is the depth-1 `Trap` event's `at` (D24). Close the set as in step 9a.f.

    d. **Steps** as in step 9a.g, with these sources: entry-block parameters → `input`; a `perform` answered from the stub log → `generated`; a `perform` answered `Live` → `provider`; every other step → `computed`. A trap in a callee ends with a `trap` step at the `Call` or `ParMap` node, `value` = `Str(<TrapKind>)`, `operands` = the replayed argument values.

    e. **Truncate and attach** as in steps 9a.h and 9a.i, on the one violation diagnostic; `tracker.free_all(&mut rt)`; add the elapsed time to `witness_us`; log `debug` `verify.witness` with `obligation` = `"t3"`.

12b. **Classify concreteness (T3)** (D43). On a T3 violation: `concrete = Some(ConcreteCheck { status: Concrete, reason: None, witness: None })`, unless `used_unbound_deterministic` was set for the violating attempt, in which case `status: Abstract`, `reason: "stubbed_operation"`. Bound `deterministic` operations ran for real and callees ran their accepted implementations; values of non-deterministic operations are environment inputs. No replay runs.

13. **Assemble.** `assumed` (accepted results only; empty otherwise) = qualified names of every `deterministic` operation that appears in a candidate `Perform`, in a `requires`/`ensures` `OpApp`, or in an `OpApp` of a callee `ensures` assumed at a call site; sorted, deduplicated; the qualified form is `<capability qualified name>::<op>` via `input.interner`. `witness_us` = `Some(<sum of the times added in steps 9a.i, 9b.f and 12a.e>)` when at least one replay ran, else `None`. `vacuity` as set in step 6a. Log `info` `verify.done` with `accepted`, `evidence`, `t2_us`, `t3_us`, `witness_us`, `diagnostics` (count). `verify` returns the result; on `Err(e)` from `try_verify` it returns `{accepted: false, evidence: None, assumed: [], diagnostics: [VerificationInconclusive, tier t2, node 0, reason <r>, message "verifier internal failure: <e>"], t2_us, t3_us: None, witness_us: None, vacuity: None}`, where `<r>` is `"unsupported_construct"` for `VerifyError::UnsupportedConstruct` and `"solver_unknown"` for every other variant (D44); it logs `error` `verify.internal` with the `VerifyError` text (fail closed; never accepts).

#### Data Model

N/A — no persistent storage; every structure lives for one `verify` call. In-memory structures:

```rust
struct Obligation { id: u32, kind: ObligationKind, script: String, node: NodeId, span: Option<Span>, clause: Option<String> }
enum ObligationKind {
    TrapSite { value: Option<ValueId>, trap: TrapKind },          // None for Trap(Unreachable)
    CallRequires { call: ValueId, callee: DefId, clause: u32 },
    Decreases { edge: u32 },
    Ensures { clause: u32 },
}
enum Outcome { Unsat, Sat(ModelValues), Unknown(&'static str) }   // reason from the closed set
struct ModelValues { inputs: Vec<(String, Value)>, expected: Vec<(String, Value)>, actual: Vec<(String, Value)> }
struct FuzzState { completed: u64, attempts: u64, rejected_by_requires: u64, discarded_resource: u64,
                   memo: HashMap<(CapId, Symbol, Vec<Value>), Value>,
                   stub_log: Vec<(CapId, Symbol, Value)>,          // (D25) stubbed results of the current attempt, in call order
                   violating_inputs: Vec<Value>, violating_trap: RtTrap,
                   violating_result: Option<Value>, violating_clause: Option<u32>,    // set on a violation (step 11d)
                   live_memo: HashMap<(CapId, Symbol, Vec<Value>), Value>,          // (D42) shared by fuzz_live and the evaluator
                   used_unbound_deterministic: bool }                               // (D43) per attempt; read by step 12b
struct T2Oracle { values: BTreeMap<ValueId, Value> }             // step 9a.a; answers depth 1 only
// T2Oracle and T3Oracle answer call_type from input.callees[callee].ret (falling back to None, then the program)
// and op_type from input.cap_ops[cap] (the CapOpSig whose name is op), so replay never meets an untyped value (D24).
struct T3Oracle<'a> { log: &'a [(CapId, Symbol, Value)], next: usize, live_ops: BTreeSet<(CapId, Symbol)> }  // step 12a.a
struct ConcreteOracle { values: BTreeMap<ValueId, Value>, gen: SplitMix64, stop: Option<&'static str>,
                        live_ops: BTreeSet<(CapId, Symbol)> }       // step 9b.b; stop: "callee_unimplemented" | "stubbed_operation"
struct ProviderThread { abandoned: Arc<AtomicBool>, in_provider: Arc<AtomicU32> }   // D41; in_provider: 0 = none, op_id + 1
static ABANDONED_THREADS: AtomicU32;                                // D41; process-wide count of abandoned threads still running; limit 8
```

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Obligation `sat` for an `ensures` clause | rejected, evidence `none` | `ContractViolation` | `{"kind":"ContractViolation","tier":"t2","node":0,"clause":"ensures: <text>","message":"ensures clause violated","counterexample":{…}}` |
| Obligation `sat` for a callee `requires` | rejected, evidence `none` | `ContractViolation` | `{"kind":"ContractViolation","tier":"t2","node":<call node>,"clause":"requires of <Callee>: <text>","message":"call precondition violated","counterexample":{…}}` |
| Trap site reachable under `requires` | rejected, evidence `none` | `ReachableTrap` | `{"kind":"ReachableTrap","tier":"t2","node":<n>,"message":"trap <Kind> reachable under requires","counterexample":{"inputs":{…},"expected":{},"actual":{"trap":"<Kind>"}}}` |
| Measure not shown to decrease on a call edge | rejected, evidence `none` | `NonDecreasing` | `{"kind":"NonDecreasing","tier":"t2","node":<edge node>,"message":"measure of <Callee> does not decrease at this call","counterexample":{…}}` |
| Measure contains `Call` or `Perform` | rejected, evidence `none` | `NonDecreasing` | `{"kind":"NonDecreasing","tier":"t2","node":<decreases node>,"message":"measure must not call specs or perform operations"}` |
| Obligation `unknown` under `assurance proved` | rejected, evidence `none` | `VerificationInconclusive` | `{"kind":"VerificationInconclusive","tier":"t2","node":<n>,"reason":"timeout"\|"budget_cut_off"\|"solver_unknown","message":"obligation <kind> #<id> not decided"}` |
| Fuzzing finds a trap | rejected, evidence `none` | `ReachableTrap` | as T2 row, `"tier":"t3"`, `"node":0` |
| Fuzzing finds a violated `ensures` | rejected, evidence `none` | `ContractViolation` | as T2 row, `"tier":"t3"` |
| Determinism probe mismatch | rejected, evidence `none` | `NonDeterministicProvider` | `{"kind":"NonDeterministicProvider","tier":"t3","node":<perform node>,"message":"deterministic operation <q> returned different results for identical arguments"}` |
| T3 deadline before `iterations` completed, with resource discards | rejected, evidence `none` | `VerificationInconclusive` | `{"kind":"VerificationInconclusive","tier":"t3","node":0,"reason":"fuzz_resource_limit","message":"fuzzing completed <c> of <n> iterations"}` |
| T3 deadline before `iterations` completed, no resource discards | rejected, evidence `none` | `VerificationInconclusive` | `…"reason":"fuzz_budget_incomplete"…` |
| A called spec has no accepted implementation and T3 is required | rejected, evidence `none` | `VerificationInconclusive` | `…"tier":"t3","reason":"callee_unimplemented","message":"fuzzing needs an accepted implementation of <Callee>"…` |
| Z3 context cannot be created | `Err` from `try_verify` | `VerifyError::SolverInit` | `"solver initialisation failed: <message>"` |
| Back edge in the block graph | `Err` | `VerifyError::CyclicBlockGraph` | `"block graph of <def> is not acyclic"` |
| Encoder meets RC instructions, `Fn`/`Token` values, or Z3 rejects a generated script | `Err` | `VerifyError::UnsupportedConstruct` | `"cannot encode <what> in <def>"` |
| Model value outside its type | `Err` | `VerifyError::ModelDecode` | `"cannot decode model of obligation <id>: <message>"` |
| `assurance tested`, T3 needed, no JIT module supplied | `Err` | `VerifyError::JitMissing` | `"fuzzing required but no JIT module was supplied"` |
| `Call` to a spec absent from `callees` | `Err` | `VerifyError::MissingCalleeSpec` | `"no spec supplied for callee <def>"` |
| Any `Err` other than `UnsupportedConstruct` reached through `verify` (not `try_verify`) | rejected, evidence `none` | `VerificationInconclusive` | `{"kind":"VerificationInconclusive","tier":"t2","node":0,"reason":"solver_unknown","message":"verifier internal failure: <error text>"}` |
| `UnsupportedConstruct` reached through `verify` (D44) | rejected, evidence `none` | `VerificationInconclusive` | `{"kind":"VerificationInconclusive","tier":"t2","node":0,"reason":"unsupported_construct","message":"verifier internal failure: cannot encode <what> in <def>"}` |
| No input satisfies `requires` (V1 `unsat`, D37) | rejected, evidence `none`; no obligation runs | `VacuousContract` | `{"kind":"VacuousContract","tier":"t2","severity":"error","node":0,"span":{…},"clause":"requires: x > 5 && x < 3","message":"no input satisfies requires"}` |
| No result satisfies `ensures` under `requires` (V2 `unsat`, D37) | rejected, evidence `none`; no obligation runs | `VacuousContract` | `{"kind":"VacuousContract","tier":"t2","severity":"error","node":0,"clause":"ensures: result == 1 && result == 2","message":"no result satisfies ensures for any input satisfying requires"}` |
| Vacuity check `unknown` (D37) | unchanged by the check | `VacuityUnknown` | `{"kind":"VacuityUnknown","tier":"t2","severity":"note","node":0,"reason":"timeout","message":"could not decide whether the contract is satisfiable"}` appended after every other diagnostic |
| Provider panic while the T3 evaluator applies a contract `OpApp` (D42) | rejected, evidence `none` | `VerificationInconclusive` | `{"kind":"VerificationInconclusive","tier":"t3","node":0,"reason":"provider_failure","message":"provider operation <q> panicked while a contract was evaluated"}` |
| T3 provider thread not finished `provider_wait_ms` after the T3 deadline (D41) | rejected, evidence `none`; thread abandoned | `VerificationInconclusive` | `{"kind":"VerificationInconclusive","tier":"t3","node":0,"reason":"provider_timeout","message":"provider operation <q> did not return within verify.provider_wait_ms"}` |
| 8 abandoned provider threads already running (D41) | T3: rejected, evidence `none`; T2: concreteness `abstract` | `VerificationInconclusive` / `concrete` | T3: `…"reason":"provider_timeout","message":"8 abandoned provider calls are still running; restart cprimed"…`; T2: `"concrete":{"status":"abstract","reason":"provider_timeout"}` |
| Concrete replay does not reproduce a T2 counterexample (D43) | rejected, evidence `none` (unchanged) | `ContractViolation` / `ReachableTrap` | the diagnostic with `"concrete":{"status":"abstract","reason":"no_violation"\|"different_outcome","witness":{…}}` (witness only when `witness_max_steps > 0`) |
| Concrete replay needs an unimplemented callee or an unbound `deterministic` operation (D43) | rejected, evidence `none` (unchanged) | `ContractViolation` / `ReachableTrap` | `"concrete":{"status":"abstract","reason":"callee_unimplemented"\|"stubbed_operation"}` |
| Concrete replay hits fuel, memory or depth, or a provider panics (D43) | rejected, evidence `none` (unchanged) | `ContractViolation` / `ReachableTrap` | `"concrete":{"status":"abstract","reason":"resource_limit"\|"provider_failure"}` |
| Replay of a T2 or T3 counterexample disagrees with the reported outcome | rejected, evidence `none` (unchanged) | `ContractViolation` / `ReachableTrap` | the diagnostic as above with `"witness":{"status":"not_reproduced","omitted":0,"steps":[…]}`; log `error` `verify.witness_mismatch` |
| Replay of a `TrapSite` or `CallRequires` model traps at another trap site first | rejected, evidence `none` (unchanged) | `ReachableTrap` / `ContractViolation` | the diagnostic as above with `"witness":{"status":"trap_first",…}` ending in a `trap` step at the site reached |
| `verify.witness_max_steps` is 0 | as without witnesses | — | no `witness` member on any diagnostic and no `concrete.witness`; `concrete` is still set (D43), so `witness_us` is `Some` when a concrete replay ran, else `None` |
| Slice longer than `verify.witness_max_steps` | rejected, evidence `none` (unchanged) | `ContractViolation` / `ReachableTrap` | `"witness":{"status":…,"omitted":<dropped count>,"steps":[<last witness_max_steps steps>]}` |
| Oracle value for a `Call`, `Perform` or `ParMap` cannot be decoded from the model | `Err` | `VerifyError::ModelDecode` | `"cannot decode model of obligation <id>: <message>"` |

#### Acceptance Criteria (Gherkin)

```gherkin
Feature: cprime-verify

  Scenario: Happy path — corrected generate_token is proved
    Given spec Auth::generate_token from concept §4.1 with assurance proved
    And the candidate body computes (+ now 3600)
    When verify runs
    Then every obligation is unsat
    And the result is accepted with evidence "proved"
    And assumed equals ["Auth::CryptoCap::hmac_sha256"]
    And t3_us is None

  Scenario: Error path — wrong expiry yields a counterexample
    Given the same spec and a candidate body computing (+ now 1000)
    When verify runs
    Then the result is rejected with evidence "none"
    And exactly one diagnostic has kind ContractViolation, tier t2 and clause "ensures: result.expires_at == now + 3600"
    And its counterexample expected value for "result.expires_at" equals its input "now" plus 3600
    And its counterexample actual value for "result.expires_at" equals its input "now" plus 1000

  Scenario: Error path — revision-1 contract reaches an overflow trap
    Given the spec without the clause "now <= U64::MAX - 3600"
    And the candidate body computes (+ now 3600)
    When verify runs
    Then a diagnostic has kind ReachableTrap, tier t2 and node equal to the node of (+ now 3600)
    And its counterexample input "now" is greater than 18446744073709548015
    And its actual map is {"trap": "Overflow"}

  Scenario: Edge case — guarded division proves no trap
    Given spec div(a, b U64) U64 with "requires b > 0" and "ensures result * b <= a"
    And the candidate body is (/ a b)
    When verify runs
    Then the DivisionByZero trap-site obligation is unsat
    And the result is accepted with evidence "proved"

  Scenario: Edge case — undefined contract clause counts as violated
    Given spec f(v Vec[U64]) U64 with "ensures result == index(v, 0)" and no requires
    And the candidate body returns 0
    When verify runs
    Then the ensures obligation is sat with input v equal to []
    And the diagnostic kind is ContractViolation

  Scenario: Error path — call precondition violated
    Given spec Math::half(x U64) U64 with "requires x % 2 == 0"
    And a candidate for spec Math::caller(y U64) U64 whose body is (Math::half y)
    When verify runs
    Then a diagnostic has kind ContractViolation and clause "requires of Math::half: x % 2 == 0"
    And its node is the node of the call form

  Scenario: Error path — mutual recursion measure does not decrease on an accepted member's edge
    Given accepted Math::even with (decreases n) calling (Math::odd n)
    And a candidate Math::odd with (decreases n) calling (Math::even (- n 1)) under n > 0
    When verify runs
    Then a NonDecreasing diagnostic names caller Math::even
    And the result is rejected

  Scenario: Error path — inconclusive under assurance proved
    Given a spec with assurance proved whose ensures needs nonlinear integer reasoning Z3 reports unknown for
    And verify.obligation_timeout_ms is 1
    When verify runs
    Then the result is rejected with evidence "none"
    And each undecided obligation has a VerificationInconclusive diagnostic with reason "timeout" or "solver_unknown"
    And no fuzzing runs

  Scenario: Edge case — budget cut-off follows the fixed order
    Given a spec with 40 ensures clauses, verify.solver_threads 1 and verify.submission_budget_ms 5
    When verify runs
    Then obligations are started in the order trap sites, call preconditions, decreases, ensures
    And every obligation not finished at the deadline is Unknown with reason "budget_cut_off"

  Scenario: Happy path — assurance tested falls back to fuzzing and accepts
    Given a spec with assurance tested whose single ensures obligation is unknown
    And a candidate that satisfies the contract on every input
    When verify runs with fuzz.iterations 10000
    Then T3 completes 10000 requires-satisfying iterations
    And the result is accepted with evidence "tested" and t3_us is Some

  Scenario: Error path — fuzzing finds a violation
    Given a spec with assurance tested and an unknown T2 obligation
    And a candidate that violates ensures when its input x equals 0
    When verify runs
    Then the result is rejected with a ContractViolation diagnostic of tier t3

  Scenario: Error path — rejection sampling starves
    Given a spec with "assurance tested" and "requires x == 123456789123" over x: U64
    And fuzz.time_budget_ms is 50
    When verify runs and fewer than fuzz.iterations inputs satisfy requires before the deadline
    Then the result is rejected with VerificationInconclusive, tier t3, reason "fuzz_budget_incomplete"

  Scenario: Error path — non-deterministic provider detected
    Given CryptoCap is bound to a provider whose deterministic operation returns a random value
    And a spec with assurance tested whose implementation performs that operation
    When the determinism probe runs
    Then the result is rejected with NonDeterministicProvider naming the operation

  Scenario: Error path — fuzzing needs an unimplemented callee
    Given a spec with assurance tested and an unknown T2 obligation
    And the candidate calls a spec with no accepted implementation
    When verify runs
    Then the result is rejected with VerificationInconclusive, tier t3, reason "callee_unimplemented"

  Scenario: Edge case — deterministic run
    Given fuzz.seed is unset
    When verify runs twice on the same submission
    Then both runs produce identical diagnostics and the same completed iteration count

  Scenario: Happy path — parallel map length is proved
    Given spec Batch::check(tokens Vec[Bytes]) Vec[Bool] with "ensures len(result) == len(tokens)"
    And its implementation (par_map Auth::verify_one tokens)
    When submit_impl runs with verify.obligation_timeout_ms 250
    Then the submission is accepted with evidence proved

  Scenario: Happy path — fact about an element from the mapped spec's ensures
    Given Auth::half(x U64) U64 with no requires and "ensures result == x / 2"
    And Batch::first(xs Vec[U64]) U64 with "requires len(xs) > 0" and "ensures result == index(xs, 0) / 2"
    And its implementation (index (par_map Auth::half xs) 0)
    When submit_impl runs
    Then the submission is accepted with evidence proved within the 2 s budget

  Scenario: Error path — wrong expiry carries a witness without the sig computation
    Given spec Auth::generate_token from concept §4.1 with assurance proved and verify.witness_max_steps 64
    And the candidate body computes (+ now 1000)
    When verify runs
    Then the ContractViolation diagnostic for "ensures: result.expires_at == now + 3600" has a witness with status "reproduced" and omitted 0
    And its steps have kinds ["param", "compute", "return"] in that order
    And the param step has node 2 and value equal to the counterexample input "now"
    And the compute step has node 6, value equal to input "now" plus 1000 and operands [input "now", 1000]
    And the return step has node 19
    And no step has node 10, 11, 13 or 16
    And every step has span None and name None

  Scenario: Error path — overflow witness ends in a trap step with operands
    Given the spec without the clause "now <= U64::MAX - 3600" and verify.witness_max_steps 64
    And the candidate body computes (+ now 3600)
    When verify runs
    Then the ReachableTrap diagnostic of node 6 has a witness with status "reproduced"
    And its first step has kind "param", node 2 and value equal to the counterexample input "now"
    And its last step has kind "trap", node 6, value "Overflow" and operands [input "now", 3600]

  Scenario: Edge case — hmac result is marked solver
    Given spec Auth::generate_token from concept §4.1 with assurance proved
    And a candidate whose msg binding is (++ (u64_be uid) (u64_be now)) and which is otherwise the corrected body
    When verify runs
    Then the ContractViolation diagnostic for the clause beginning "ensures: result.sig ==" has a witness with status "reproduced"
    And its witness contains exactly one step of kind "perform", with node 16 and source "solver"
    And that step's value equals the sig field of the final return step's value

  Scenario: Error path — fuzzing witness marks generated values and reproduces
    Given a spec with assurance tested and an unknown T2 obligation
    And CapId RandomCap bound to host::Random, whose random_u64 operation is not deterministic and is therefore stubbed in T3
    And a candidate that performs RandomCap::random_u64 and violates ensures when the generated value is 0
    And fuzz.seed 1
    When verify runs
    Then the result is rejected with a ContractViolation diagnostic of tier t3
    And its witness has status "reproduced"
    And its perform step has source "generated" and value equal to the first entry of the violating attempt's stub log
    And its param steps have source "input" and values equal to the counterexample inputs

  Scenario: Edge case — trap reached first on a non-dominating path
    Given spec Math::f(a, b U64) U64 with "requires a > 5" and "requires b < 3" and no ensures
    And the candidate body (let [q (if (> a 5) (/ 10 b) 0)] (- q 1))
    When verify runs
    Then the trap-site obligation of (- q 1) is sat with input b equal to 0
    And its ReachableTrap diagnostic has a witness with status "trap_first"
    And that witness's last step has kind "trap", value "DivisionByZero" and the node of (/ 10 b)
    And the result also has a ReachableTrap diagnostic whose actual map is {"trap": "DivisionByZero"}

  Scenario: Error path — model disagreeing with execution is not_reproduced
    Given spec Auth::generate_token from concept §4.1 and the candidate body computing (+ now 1000)
    And a unit test that runs step 9a for ensures#1 with model_ret.expires_at set to input "now" plus 999
    When the witness is built
    Then its status is "not_reproduced"
    And the result is rejected with evidence "none"
    And exactly one error log event "verify.witness_mismatch" is written with kind "Ensures"

  Scenario: Edge case — 100-step slice truncated to 64 with omitted 36
    Given spec Chain::add(x U64) U64 with "requires x <= 1000" and "ensures result == x + 98"
    And a candidate binding x1 = (+ x 1) and each xk = (+ xk-1 1) for k = 2 … 99, with body x99
    And verify.witness_max_steps 64
    When verify runs
    Then the ContractViolation diagnostic has a witness with 64 steps and omitted 36
    And its last step has kind "return" and value equal to input "x" plus 99
    And no step has kind "param"

  Scenario: Edge case — witness_max_steps 0 yields no witness
    Given spec Auth::generate_token from concept §4.1 and the candidate body computing (+ now 1000)
    And verify.witness_max_steps 0
    When verify runs
    Then the ContractViolation diagnostic has no witness member
    And witness_us is Some, from the concrete replay of step 9b

  Scenario: Error path — unsatisfiable requires is a vacuous contract (RFC 07 criterion 1)
    Given spec Math::f(x U64) U64 with "requires x > 5" and "requires x < 3" and "ensures result == x"
    And any type-correct candidate, for example the body x
    When verify runs with input.vacuity None
    Then the result is rejected with evidence "none"
    And exactly one diagnostic has kind VacuousContract, severity error and clause "requires: x > 5 && x < 3"
    And no diagnostic has kind ContractViolation
    And no obligation of step 7 was solved
    And result.vacuity is Vacuous

  Scenario: Error path — contradictory ensures is a vacuous contract (RFC 07 criterion 2)
    Given spec Math::g(x U64) U64 with "ensures result == 1" and "ensures result == 2"
    When verify runs on any candidate
    Then the result is rejected with one VacuousContract diagnostic whose clause is "ensures: result == 1 && result == 2"

  Scenario: Happy path — a satisfiable contract still proves (RFC 07 criterion 2)
    Given spec Auth::generate_token from concept §4.1 and the corrected body
    When verify runs with input.vacuity None
    Then result.vacuity is Satisfiable
    And the result is accepted with evidence "proved" and no diagnostics

  Scenario: Edge case — undecided vacuity check is a note (RFC 07 criterion 3)
    Given spec Auth::generate_token from concept §4.1 and the corrected body
    And verify.obligation_timeout_ms is 1 and the solver reports unknown for V1
    When verify runs
    Then the result's accepted flag is the same as with the vacuity check disabled
    And its last diagnostic has kind VacuityUnknown, severity note and a reason in {"timeout", "solver_unknown", "budget_cut_off"}

  Scenario: Edge case — a cached outcome adds no solver call (RFC 07 criterion 4)
    Given spec Auth::generate_token and the corrected body
    And input.vacuity is Some(Satisfiable)
    When verify runs with a counting solver hook
    Then no vacuity script is solved and result.vacuity is None

  Scenario: Edge case — contract evaluation calls a provider only for new arguments (RFC 07 criterion 8, restated)
    Given a test provider bound to capability Rec whose deterministic operation f(x U64) U64 records each call
    And spec Rec::s(x U64) U64 with assurance tested, "ensures result == Rec::f(x)" and an ensures obligation T2 reports unknown
    And the candidate body (perform Rec::f x)
    When verify runs with fuzz.iterations 100
    Then the number of recorded calls equals the number of distinct x the program performed f on
    And no recorded call has an argument the program did not use

  Scenario: Error path — provider panic during contract evaluation (RFC 07 criterion 9)
    Given a bound deterministic operation Rec::boom whose provider panics for every input
    And spec Rec::t(x U64) U64 with assurance tested, an unknown T2 obligation and "ensures result == Rec::boom(x)"
    And a candidate that does not perform Rec::boom
    When verify runs
    Then the result is rejected with VerificationInconclusive, tier t3, reason "provider_failure"
    And verify returns without a crash

  Scenario: Error path — a provider call that never returns is abandoned (RFC 07 criterion 10)
    Given a bound deterministic operation Rec::hang whose provider sleeps forever
    And spec Rec::u(x U64) U64 with assurance tested and an unknown T2 obligation, and a candidate that performs Rec::hang
    And fuzz.time_budget_ms 200 and verify.provider_wait_ms 100
    When verify runs
    Then it returns within 1 s with VerificationInconclusive, tier t3, reason "provider_timeout" naming Rec::hang
    And a second verify call on another spec completes normally

  Scenario: Error path — a counterexample that relies on a model-only provider value is abstract (RFC 07 criterion 12)
    Given spec Auth::sign(secret Bytes, m Bytes) Hash with "requires len(secret) >= 32" and "ensures result != x\"0000000000000000000000000000000000000000000000000000000000000000\""
    And CryptoCap bound to host::Crypto, and the candidate body (perform CryptoCap::hmac_sha256 secret m)
    When verify runs
    Then the ContractViolation's counterexample sets the hmac_sha256 result to 32 zero bytes
    And its concrete member has status "abstract" and reason "no_violation"
    And concrete.witness ends in a return step whose value is not 32 zero bytes

  Scenario: Error path — a reproducible counterexample is concrete (RFC 07 criterion 13)
    Given spec Auth::generate_token from concept §4.1 and the candidate body computing (+ now 1000)
    When verify runs
    Then the ContractViolation's concrete member is {"status": "concrete"} with no reason and no witness

  Scenario: Edge case — an unimplemented callee makes a counterexample abstract
    Given spec Math::caller(y U64) U64 with "ensures result == y" calling Math::helper, which has no accepted implementation
    And a candidate (Math::helper y) where Math::helper's ensures leaves result unconstrained
    When verify runs
    Then the ContractViolation's concrete member has status "abstract" and reason "callee_unimplemented"

  Scenario: Error path — unsupported construct through verify (RFC 07 criterion 14)
    Given a test that passes a function containing RcDup to verify
    When verify runs
    Then the single diagnostic is VerificationInconclusive with reason "unsupported_construct"
```

#### Performance, Security, Observability

- **Performance targets:** encoding (steps 2–7) ≤ 5 ms p95 for a candidate of ≤ 200 IR instructions; T2 ≤ 250 ms per obligation and ≤ 2 s per submission at defaults (enforced by Z3 `timeout` and the coordinator deadline, not merely measured); T3 10,000 iterations ≤ 5 s at defaults (enforced by the deadline); peak memory per Z3 context ≤ 512 MiB (measured in the Phase 4 corpus run; exceeding it is reported, not enforced). `t2_us` overshoot past the budget ≤ 50 ms (interrupt latency). Vacuity check (step 6a) ≤ 250 ms per script (enforced by the obligation timeout) and run at most once per spec revision per daemon when decided (D38). Concrete replay (step 9b) bounded by `fuzz.call_fuel`, `fuzz.call_memory_bytes` and `verify.provider_wait_ms`, measured through `witness_us`. Witness construction (steps 9a and 12a) ≤ 20 ms p95 per trace on the corpus, measured and reported through `witness_us` (`timings_us.witness`), not enforced; each replay is bounded by `fuzz.call_fuel` and `fuzz.call_memory_bytes`, and a T2 replay executes only the candidate's own activation, whose block graph is acyclic.
- **Security:** fuzzing performs no real I/O: only `deterministic` operations with a binding reach provider code (precondition 3, SA-30); every other operation is a stub. Fuzzed calls run with fuel and a live-heap limit; every allocation is released with `free_all`. No input is read from disk or network; Z3 receives only generated scripts. Counterexample values are synthetic (HLD data lifecycle). A `requires` that no input satisfies, or `ensures` clauses no result satisfies, is rejected by step 6a (ADR-052); contract quality beyond satisfiability remains the human's responsibility (HLD risk "contract incompleteness"). Provider code runs during checking only for bound `deterministic` operations, which must be effect-free (ADR-054), and only on a provider thread that is abandoned rather than waited on forever (D41).
- **Observability:** log events (JSON on stderr via the cross-cutting Logging spec): `verify.start` (info: `spec`, `impl_def`, `assurance`), `verify.obligations` (debug: counts per kind), `verify.obligation` (debug: `id`, `kind`, `outcome`, `duration_us`; trace: `script`), `verify.fuzz` (info: `completed`, `rejected_by_requires`, `discarded_resource`, `outcome`), `verify.witness` (debug: `obligation`, `status`, `steps`, `omitted`, `duration_us`), `verify.witness_mismatch` (error: `obligation`, `kind`, `outcome`), `verify.vacuity` (debug: `outcome`, `scripts`, `duration_us`), `verify.concrete` (debug: `obligation`, `status`, `reason`, `duration_us`), `verify.provider_abandoned` (warn: `phase` (`t3` or `concrete`), `operation` or null, `abandoned` (count)), `verify.done` (info: `accepted`, `evidence`, `t2_us`, `t3_us`, `witness_us`, `diagnostics`), `verify.internal` (error: error text). Counterexample values and witness step values appear only at `trace` (SA-26); `verify.witness` and `verify.witness_mismatch` carry no step value. Metrics: `t2_us`, `t3_us`, `witness_us` in the result, reported by `cprime-daemon` as `timings_us.t2`/`t3`/`witness`.

#### Gaps

None.
