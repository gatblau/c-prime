# Component spec — cprime-effects

> **Spec set:** cprime
> **Status:** Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> **Last updated:** 2025-12-20

### SPEC: cprime-effects
**File:** `crates/cprime-effects/src/lib.rs` (modules `effects.rs`, `termination.rs`, `bindings.rs`) | **Package:** `cprime-effects` | **Phase:** 1 | **Dependencies:** `cprime-core`, `cprime-types` (workspace); `salsa` (third-party); dev-dependency `cprime-syntax`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprime-effects` enforces the effect discipline of ADR-005 and the termination rules of ADR-024 at T0, and validates `bind` declarations against linked providers (ADR-021). It rejects an implementation whose transitive effect set is not a subset of its spec's `uses` (`UnauthorizedEffect`), rejects recursion that carries neither a `decreases` measure nor `Diverge` (`MissingDecreases`), and computes the recursive component and call edges from which `cprime-verify` generates `decreases` obligations. It also produces the capability → provider map used by fuzzing (SA-30) and the release build.

#### Approach

All three checks are syntactic walks over the typed AST and the namespace; none needs an SMT solver. The effect set uses each callee's *declared* `uses` rather than an inferred set, so checking one implementation never requires reading another's body. The recursive component is found by a forward walk from the candidate through accepted implementations followed by Tarjan's strongly-connected-components algorithm on that reachable subgraph; any cycle through the candidate lies inside the subgraph, so the rest of the project is never scanned. Rejected alternatives: inferring effect sets from callee bodies (would make a caller's acceptance depend on a callee's current body rather than its contract, breaking modular verification, SA-18); recomputing a whole-project call graph per request (cost grows with project size, A-6).

#### Shared Context

Copied from `phase-2-architecture.md` §2C and §2E.

```rust
// cprime_core::ids
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct Symbol(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct DefId(pub [u8; 32]);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct CapId(pub u32);   // CapId(0) = Diverge
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, serde::Serialize)] #[serde(transparent)] pub struct NodeId(pub u32);

// cprime_core::ir
pub enum Type { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<Type>), Struct(DefId), Sum(DefId),
                Fn { params: Vec<Type>, ret: Box<Type>, effects: BTreeSet<CapId> }, Token }

// cprime_core::tast
pub struct TSpec { pub def: DefId, pub name: String, pub params: Vec<(Symbol, Type)>, pub ret: Type,
                   pub uses: BTreeSet<CapId>, pub assurance: Assurance, pub waivers: Vec<Waiver>,
                   pub requires: Vec<TClause>, pub ensures: Vec<TClause>, pub is_entry: bool }
pub struct TImpl { pub spec: DefId, pub params: Vec<(Symbol, NodeId)>, pub decreases: Option<TDecreases>, pub body: TExpr, pub node_count: u32 }
pub struct TDecreases { pub node: NodeId, pub measure: Vec<TExpr> }
pub struct TExpr { pub node: NodeId, pub ty: Type, pub kind: TExprKind }
pub enum TExprKind {
    Const(Const), Local(u32), Let { value: Box<TExpr>, body: Box<TExpr> },
    If { cond: Box<TExpr>, then_: Box<TExpr>, else_: Box<TExpr> },
    Bin { op: BinOp, lhs: Box<TExpr>, rhs: Box<TExpr> }, Not(Box<TExpr>),
    And(Box<TExpr>, Box<TExpr>), Or(Box<TExpr>, Box<TExpr>),
    Call { func: DefId, args: Vec<TExpr> },
    Perform { cap: CapId, op: Symbol, deterministic: bool, args: Vec<TExpr> },
    Intrinsic { op: IntrinsicOp, args: Vec<TExpr> }, Len(Box<TExpr>), Index { vec: Box<TExpr>, index: Box<TExpr> },
    MakeStruct { ty: DefId, fields: Vec<TExpr> }, GetField { value: Box<TExpr>, index: u32 },
    MakeVariant { ty: DefId, tag: u32, fields: Vec<TExpr> },
    Match { scrutinee: Box<TExpr>, arms: Vec<TArm> }, MakeVec { elem: Type, items: Vec<TExpr> },
    ParMap { func: DefId, xs: Box<TExpr> },
}
pub struct TArm { pub tag: u32, pub binders: u32, pub body: TExpr }

/// Produced by this crate; consumed by cprime-verify and cprime-daemon.
pub struct TerminationInfo {
    pub component: BTreeSet<DefId>,         // recursive component containing the candidate; empty if not recursive
    pub diverges: bool,                      // candidate's uses contains Diverge: no decreases obligations
    pub edges: Vec<CallEdge>,                // every call edge inside the component
    pub diagnostics: Vec<Diagnostic>,        // MissingDecreases, TypeMismatch (measure length)
}
pub struct CallEdge { pub caller: DefId, pub callee: DefId, pub node: NodeId, pub caller_is_candidate: bool }

// cprime_core::ast (bind declarations, entries)
pub struct Ident { pub name: String, pub span: Span }
pub struct Path { pub segments: Vec<Ident>, pub span: Span }
pub struct Module { pub file: String, pub path: Path, pub items: Vec<Item> }
pub enum Item { Type(TypeDef), Capability(CapabilityDef), Spec(SpecDef), Entry(EntryDef), Bind(BindDef), Server(ServerDef) }
pub struct EntryDef { pub name: Ident, pub params: Vec<Param>, pub ret: TypeExpr, pub uses: Vec<Path>, pub span: Span }
pub struct BindDef { pub capability: Path, pub provider: Path, pub span: Span }
pub struct ServerDef { pub name: Ident, pub handler: Path, pub transport: Path, pub span: Span }   // ADR-051; handler: spec path, transport: provider path

// cprime_core::db
pub struct SourceFile { pub path: String, pub text: String }        // Salsa input
pub struct ProjectInputs { pub files: Vec<SourceFile>, pub config: ProjectConfig, pub providers: Vec<ProviderInfo> }
pub enum BoundaryTy { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<BoundaryTy>), Result(Box<BoundaryTy>, Box<BoundaryTy>) }
pub struct ProviderOpInfo { pub name: String, pub params: Vec<BoundaryTy>, pub ret: BoundaryTy, pub deterministic: bool, pub flat_index: u32,
    pub vectors: u32 }   // (D40) not read by this crate
pub struct ProviderInfo { pub path: String, pub crate_name: String, pub crate_version: String, pub ops: Vec<ProviderOpInfo>, pub thread_safe: bool, pub transport: Option<TransportInfo> }
#[derive(Clone, PartialEq, Eq, Hash)]
pub struct TransportInfo { pub version: u32, pub request: Vec<(String, BoundaryTy)>, pub response: Vec<(String, BoundaryTy)> }   // ADR-051, SA-66
pub trait Db: salsa::Database {
    fn inputs(&self) -> ProjectInputs; fn interner(&self) -> &Interner;
    fn parse_module(&self, file: SourceFile) -> Arc<ParsedModule>;
    fn accepted_impl(&self, spec: DefId) -> Option<Arc<AcceptedImpl>>;
    fn impl_status(&self, spec: DefId) -> ImplStatus;
}
pub struct ParsedModule { pub module: Option<ast::Module>, pub diagnostics: Vec<Diagnostic> }
pub struct AcceptedImpl { pub file: String, pub header: AcceptanceHeader, pub timpl: TImpl, pub function: ir::Function }

// cprime_core::namespace / resolve
pub struct NamespaceTable { pub entries: BTreeMap<String, NsEntry> }
pub struct NsEntry { pub def: DefId, pub kind: NsKind, pub file: String, pub span: Span }
pub enum NsKind { Type, Capability, Spec, Entry, Server, Variant { sum: DefId, tag: u32 } }
pub enum DefKind { Type, Capability, Spec, Entry, Prelude }
pub struct Scope<'a> { pub module: &'a str, pub locals: &'a [Symbol] }
pub enum Res { Local(u32), Def { def: DefId, kind: DefKind }, Variant { sum: DefId, tag: u32 },
               CapOp { cap: CapId, cap_def: DefId, op: Symbol }, Prelude(PreludeItem), Builtin(BuiltinType) }
pub fn namespace_table(db: &dyn Db) -> Arc<NamespaceTable>;
pub fn lookup(db: &dyn Db, scope: &Scope, path: &Path) -> Result<Res, Diagnostic>;

// cprime_types (called)
pub struct SpecCheck { pub spec: TSpec, pub schema: TypeSchema, pub diagnostics: Vec<Diagnostic> }
pub struct CapOpSig { pub name: Symbol, pub params: Vec<Type>, pub ret: Type, pub deterministic: bool }
pub fn check_spec(db: &dyn Db, def: DefId) -> Arc<SpecCheck>;
pub fn capability_ops(db: &dyn Db, cap: CapId) -> Arc<Vec<CapOpSig>>;
pub fn mono_instance_id(generic: DefId, args: &[Type]) -> DefId;  // instance DefId of a generic (here: prelude Result)

// cprime_core::diag
pub enum DiagnosticKind { ParseError, UnknownName, ArityMismatch, TypeMismatch, UnauthorizedEffect, MissingDecreases,
    NonExhaustiveMatch, UnconstrainedOutput, InvalidParMap, ContractViolation, ReachableTrap, NonDecreasing,
    VerificationInconclusive, NonDeterministicProvider, InvalidServer, VacuousContract, VacuityUnknown }   // (D37)
pub enum Tier { T0, T1, T2, T3 }
pub enum Severity { Error, Note }   // (D37) Note only on VacuityUnknown
pub struct Span { pub file: String, pub line: u32, pub column: u32 }
pub struct Diagnostic { pub kind: DiagnosticKind, pub tier: Tier, pub severity: Severity, pub node: NodeId,
    pub span: Option<Span>, pub message: String, pub clause: Option<String>,
    pub counterexample: Option<Counterexample>, pub reason: Option<String>,
    pub witness: Option<WitnessTrace>,    // (D22) WitnessTrace is defined in cprime_core::diag; always None for diagnostics this crate produces
    pub concrete: Option<ConcreteCheck> } // (D43) defined in cprime_core::diag; always None for diagnostics this crate produces
```

Rules carried in: a project declares at most one `entry`; `bind` is legal only in the module that declares the `entry` (SA-29). Built-in providers `host::Crypto`, `host::Clock`, `host::Random`, `host::Log` are always present in `ProjectInputs.providers`, in that order, followed by `cprime.toml` providers (SA-09). Measures are tuples of `U64` expressions compared lexicographically (ADR-024); one `decreases` obligation per call edge inside the component, including edges inside already-accepted members (SA-20).

**Parallel map (RFC 02, D16).** At each `TExprKind::ParMap { func: S, .. }`: `Diverge ∈ S.uses` → `InvalidParMap`; for each other `C ∈ S.uses`, `C` unbound in `check_bindings().bindings`, its provider's `ProviderInfo.thread_safe = false`, or some operation of `capability_ops(C)` not `deterministic` → `InvalidParMap`. The form adds `S.uses` to the transitive effect set as a call does, and the edge caller → `S` to the call graph.

#### Public Interface

```rust
/// UnauthorizedEffect diagnostics for `imp`; empty when its transitive effect set ⊆ spec.uses.
pub fn check_effects(db: &dyn Db, spec: &TSpec, imp: &TImpl) -> Vec<Diagnostic>;

/// Recursive component of the candidate, its call edges, and termination-structure diagnostics.
pub fn termination(db: &dyn Db, spec: &TSpec, imp: &TImpl) -> TerminationInfo;

pub struct BindingCheck {
    pub bindings: BTreeMap<CapId, u32>,     // capability → index into ProjectInputs.providers (valid binds only)
    pub diagnostics: Vec<Diagnostic>,
}
/// Validates entry uniqueness, bind placement and bind signatures for the whole project. Salsa-tracked.
pub fn check_bindings(db: &dyn Db) -> Arc<BindingCheck>;
```

`check_bindings` returns `Arc<BindingCheck>` because it is a tracked query (§2E, D11). Callers dereference.

Consumers: `cprime-daemon` calls `check_effects` and `termination` in T0 for every `check_impl`/`submit_impl`, passes `TerminationInfo` to `cprime-verify`, and uses `check_bindings().bindings` only (its diagnostics are not reported by the daemon). `cprime-cli` calls all three during release builds and treats any diagnostic from any of them as fatal.

##### Example

```text
module Math
spec even(n U64) Bool {
    ensures result == (n % 2 == 0)
}
spec odd(n U64) Bool {
    ensures result == (n % 2 == 1)
}
```
Accepted `math/odd.opq`: `(impl Math::odd [n] (decreases n) (if (== n 0) false (Math::even (- n 1))))`.
Candidate `math/even.opq`: `(impl Math::even [n] (decreases n) (if (== n 0) true (Math::odd (- n 1))))`.

`termination` returns:

```text
component = { DefId(Math::even), DefId(Math::odd) }
diverges  = false
edges = [
  CallEdge { caller: even, callee: odd,  node: NodeId(9), caller_is_candidate: true  },   // pre-order: impl 0, n 1, decreases 2, n 3, if 4, == 5, n 6, 0 7, true 8, call 9
  CallEdge { caller: odd,  callee: even, node: NodeId(9), caller_is_candidate: false },   // node in odd.opq
]
diagnostics = []
```

The same candidate without `(decreases n)` yields:

```json
{"kind":"MissingDecreases","tier":"t0","severity":"error","node":0,
 "message":"`Math::even` is recursive (component: Math::even, Math::odd) and has neither a decreases clause nor Diverge in its uses list"}
```

`check_effects` on an implementation of `spec f() U64 {}` (no `uses` line) that performs `Net::send` (with `capability Net { send(b Bytes) }`) yields `{"kind":"UnauthorizedEffect","node":3,"message":"`Main::Net` is performed but is not in the uses list of `Main::f`", …}`.

#### Internal Logic

All diagnostics: `tier: T0`, `severity: Error`. Diagnostics about the submission carry the offending node and its span; diagnostics about `.cprime` declarations carry `node: NodeId(0)` and a span into the `.cprime` file (SA-32). Capability display names come from the interner: the qualified name (`Auth::CryptoCap`), or `Diverge` for `CapId(0)`. Each public function logs one `debug` line on exit (target `cprime_effects`, fields `fn`, `spec`, `diagnostics`, `duration_us`); no source text is logged.

**E — `check_effects(db, spec, imp)`**

E-1. Walk `imp.body` (and `imp.decreases` measures) in pre-order. For each `Perform{cap}` record `(cap, node, "performed")` the first time `cap` is seen. For each `Call{func}` or `ParMap{func}` call `cprime_types::check_spec(db, func)` and, for every capability in the callee's `uses` (including `CapId(0)`), record `(cap, node, "via <callee name>")` the first time `cap` is seen.
E-PM. At each `ParMap{func: S}` (D16), in this order: `Diverge ∈ S.uses` → `InvalidParMap` at the `ParMap` node "`<S>` may diverge; `par_map` needs a terminating spec". Then for each `C ∈ S.uses` other than `Diverge`, in `CapId` order: `C` absent from `check_bindings(db).bindings`, or `ProjectInputs.providers[bindings[C]].thread_safe == false`, or any `capability_ops(C)` entry with `deterministic == false` → one `InvalidParMap` at the node "`<S>` uses `<C>`, which is not deterministic and thread-safe; `par_map` needs both".
E-2. The transitive effect set is the set of recorded capabilities. For each recorded capability not in `spec.uses`, emit `UnauthorizedEffect` at the recorded node: performed → "`<Cap>` is performed but is not in the uses list of `<spec>`"; via a call → "call to `<callee>` requires `<Cap>`, which is not in the uses list of `<spec>`". For `Diverge` via a call → "call to `<callee>` may diverge (Diverge), which is not in the uses list of `<spec>`".
E-3. Diagnostics are ordered by node. Entries follow the same rule against the entry's `uses`. No other error is possible: `check_impl` has already guaranteed every callee and capability resolves.

**T — `termination(db, spec, imp)`**

T-1. Candidate edges: every `Call{func}` and `ParMap{func}` in `imp.body` in pre-order, as `(spec.def, func, node, true)`.
T-2. Forward exploration: worklist starting from the callees of the candidate. For each spec `s ≠ spec.def` popped, call `Db::accepted_impl(s)`; when it returns `Some(a)`, collect `a.timpl`'s calls and `ParMap`s as `(s, callee, node, false)` and push unseen callees. A spec with no accepted implementation has no outgoing edges. The candidate's own spec is never read through `accepted_impl` (the candidate replaces it). Reachable set `R` = the candidate plus every popped spec.
T-3. Run Tarjan's algorithm on the subgraph over `R`. Let `C` be the strongly connected component containing `spec.def`. The candidate is **recursive** when `|C| > 1` or a candidate edge targets `spec.def`.
T-4. Not recursive → return `component: {}`, `edges: []`, `diverges: spec.uses ∋ CapId(0)`, `diagnostics: []`. A `decreases` clause on a non-recursive implementation is allowed and produces no obligation.
T-5. Recursive → `component = C`; `diverges = spec.uses ∋ CapId(0)`. For each member `m ∈ C` obtain its `TSpec` (`check_spec`) and its `TImpl` (the candidate for `spec.def`, `accepted_impl(m).timpl` otherwise).
T-6. `MissingDecreases` rule — each member without `Diverge` in its `uses` must carry `decreases`:
  - Candidate lacks both → `MissingDecreases` at node 0: "`<spec>` is recursive (component: <sorted member names>) and has neither a decreases clause nor Diverge in its uses list".
  - An accepted member lacks both → `MissingDecreases` at node 0: "accepted implementation of `<m>` joins the recursive component of `<spec>` but has no decreases clause; resubmit `<m>` with one".
T-7. Measure length — among members without `Diverge` that carry `decreases`, the reference length is the candidate's measure length when it has one, else the first such member in `DefId` order. Every other member's length must match: the candidate differs → `TypeMismatch` at the candidate's `decreases` node "decreases measure has <k> element(s); the recursive component uses <n>"; an accepted member differs → `TypeMismatch` at node 0 (or the candidate's `decreases` node when present) "decreases measure of accepted `<m>` has <k> element(s); `<spec>` uses <n>".
T-8. Mixed components: a member with `Diverge` cannot be called from a member without it, because `check_effects` (E-2) rejects such a call. A component therefore contains either only `Diverge` members or none; no extra rule is needed. Edges whose caller has `Diverge` are excluded from `edges` (no obligation).
T-9. `edges` = every collected edge with `caller ∈ C`, `callee ∈ C` and caller without `Diverge`. Order: candidate edges in pre-order, then accepted members' edges sorted by `(caller DefId, node)`. Node ids of non-candidate edges refer to that member's own `.opq` file.
T-10. Return `TerminationInfo`. The `NonDecreasing` diagnostic is not produced here; `cprime-verify` emits it when an edge's obligation fails.

**B — `check_bindings(db)`**

B-1. Walk every `.cprime` file in `ProjectInputs.files` through `Db::parse_module` (files with parse diagnostics are skipped; their errors are reported by the stage that parses them). Collect all `EntryDef`s, `ServerDef`s (ADR-051) and `BindDef`s with their module path. The single entry or server is the program *root*.
B-2. More than one root (entries and servers together) → `TypeMismatch` at the second and later roots' spans: "project declares more than one entry: `<first>`, `<this>`" (both kinds count; ADR-051).
B-3. A `bind` outside the entry's module, or any `bind` when no entry exists → `TypeMismatch` "bind declarations are allowed only in the module that declares the entry (`<entry module>`)" / "bind declarations require an entry in the same module". Here "entry" means the root, entry or server.
B-4. For each remaining `bind`: resolve `capability` with `lookup(Scope{module: <entry module>, locals: []})`. Failure → the `UnknownName` from `lookup`. `Prelude(Diverge)` → `TypeMismatch` "Diverge cannot be bound". Any non-capability → `TypeMismatch` "`<path>` is not a capability".
B-5. Join `provider` segments with `::` and find the first `ProviderInfo` with that `path`. None → `UnknownName` "no provider `<path>` is linked; linked providers: <comma-separated paths>".
B-6. A capability bound twice → `TypeMismatch` at the second bind "capability `<Cap>` is already bound to `<provider>`".
B-7. Signature check: for every `CapOpSig` of `capability_ops(cap)` find the provider op with the same name; missing → `TypeMismatch` "provider `<P>` has no operation `<op>` required by `<Cap>`". Parameter counts differ → `TypeMismatch` "operation `<op>`: `<Cap>` declares <k> parameter(s), provider `<P>` declares <m>". Each parameter and the result must satisfy `matches(Type, BoundaryTy)`: `Unit`↔`Unit`, `Bool`↔`Bool`, `U8`↔`U8`, `U64`↔`U64`, `I64`↔`I64`, `Bytes`↔`Bytes`, `FixedBytes(n)`↔`FixedBytes(n)`, `Vec(t)`↔`Vec(b)` when `matches(t, b)`, `Sum(d)`↔`Result(bt, be)` when `d == mono_instance_id(<prelude Result DefId>, [t, e])` for the types `t`, `e` that match `bt`, `be`; nothing else matches. Mismatch → `TypeMismatch` "operation `<op>`: parameter <i> is `<contract type>` in `<Cap>` but `<boundary type>` in provider `<P>`" (or "result is …"). `deterministic` differs → `TypeMismatch` "operation `<op>` is <deterministic|not deterministic> in `<Cap>` but <not deterministic|deterministic> in provider `<P>`". Provider operations not named by the capability are allowed.
B-8. A bind with any B-4…B-7 diagnostic is excluded from `bindings`; otherwise `bindings[cap] = provider index`.
B-9. Completeness for release: for each capability in the entry's `uses` other than `Diverge` without a valid bind → `TypeMismatch` at the entry's span "capability `<Cap>` in the uses list of entry `<entry>` has no bind". (`cprime-daemon` ignores these diagnostics; `cprime-cli` treats them as fatal.) For a server root the list is the handler spec's `uses`, and the message names the handler: "capability `<Cap>` in the uses list of handler `<Spec>` has no bind".
B-11. Server root (ADR-051, SA-65, D32), in this order, each failure an `InvalidServer` diagnostic at the server's span:
  - (a) resolve `handler` with `lookup(Scope{module: <server module>, locals: []})`; not a spec → "`<path>` is not a spec; a server handler must be a spec";
  - (b) the handler spec has exactly one parameter → else "handler `<Spec>` takes <n> parameters; a handler takes exactly one request";
  - (c) no `requires` clause → else "handler `<Spec>` has a requires clause; a handler receives untrusted input and cannot have one";
  - (d) `Diverge ∉ uses` → else "handler `<Spec>` may diverge; a handler must terminate";
  - (e) every capability in `uses` bound (B-9) to a provider with `thread_safe = true` → else "handler `<Spec>` uses `<Cap>`, bound to `<P>`, which is not thread-safe";
  - (f) find the `ProviderInfo` whose `path` is the `transport` text; none → `UnknownName` as B-5; `transport = None` → "provider `<P>` is not a transport"; `thread_safe = false` → "transport `<P>` is not thread-safe"; `version ≠ 1` → "transport `<P>` uses interface version <v>; this compiler supports 1";
  - (g) the parameter type is a struct whose fields equal `transport.request` (same count; per position same name and a field type equal to the boundary type under the B-7 mapping) → else "handler `<Spec>` request type `<T>` does not match transport `<P>`: field <i> is `<name> <type>`, expected `<name> <type>`"; the result type likewise against `transport.response`, with "response type".
B-10. Diagnostics are ordered by (file, line, column).

#### Data Model

N/A — no persistent storage. In-memory only: the reachable call subgraph (`BTreeMap<DefId, Vec<(DefId, NodeId)>>`) built per `termination` call and discarded; `BindingCheck` memoised by Salsa until a `.cprime` file or the provider list changes.

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Implementation performs a capability not in `uses` | rejected (T0) | `UnauthorizedEffect` | `"message":"`Main::Net` is performed but is not in the uses list of `Main::f`"` |
| Implementation calls a spec whose `uses` exceeds the caller's | rejected (T0) | `UnauthorizedEffect` | `"message":"call to `Io::read` requires `Io::Disk`, which is not in the uses list of `Main::f`"` |
| Call to a spec granted `Diverge` from a spec without it | rejected (T0) | `UnauthorizedEffect` | `"message":"call to `Srv::loop` may diverge (Diverge), which is not in the uses list of `Main::f`"` |
| Recursive candidate without `decreases` or `Diverge` | rejected (T0) | `MissingDecreases` | see Example |
| Accepted member of the new component lacks `decreases` | rejected (T0) | `MissingDecreases` | `"message":"accepted implementation of `Math::odd` joins the recursive component of `Math::even` but has no decreases clause; resubmit `Math::odd` with one"` |
| Measure tuple lengths differ inside a component | rejected (T0) | `TypeMismatch` | `"message":"decreases measure has 2 element(s); the recursive component uses 1"` |
| Server handler or transport invalid (B-11) | binding diagnostic (fatal for release) | `InvalidServer` | `"message":"handler `Api::handle` uses `Store`, bound to `acme::Redis`, which is not thread-safe"` |
| More than one entry in the project | binding diagnostic | `TypeMismatch` | `"message":"project declares more than one entry: `Main::main`, `Tool::main`"` |
| `bind` outside the entry module, or no entry | binding diagnostic | `TypeMismatch` | `"message":"bind declarations are allowed only in the module that declares the entry (`Main`)"` |
| Bound capability or provider path unknown | binding diagnostic | `UnknownName` | `"message":"no provider `acme::Zstd` is linked; linked providers: host::Crypto, host::Clock, host::Random, host::Log"` |
| Capability bound twice / `Diverge` bound | binding diagnostic | `TypeMismatch` | `"message":"capability `Main::CryptoCap` is already bound to `host::Crypto`"` |
| Provider lacks an op, or signature / `deterministic` differs | binding diagnostic | `TypeMismatch` | `"message":"operation `hmac_sha256`: parameter 2 is `Bytes` in `Main::CryptoCap` but `FixedBytes[32]` in provider `host::Crypto`"` |
| Entry `uses` capability without bind | binding diagnostic (fatal for release) | `TypeMismatch` | `"message":"capability `Main::CryptoCap` in the uses list of entry `Main::main` has no bind"` |
| Mapped spec uses `Diverge` | rejected (T0) | `InvalidParMap` | ``"message":"`Net::poll_one` may diverge; `par_map` needs a terminating spec"`` |
| Mapped spec uses a capability with a non-deterministic operation, an unbound capability, or one whose provider is not thread-safe | rejected (T0) | `InvalidParMap` | ``"message":"`Audit::stamp` uses `ClockCap`, which is not deterministic and thread-safe; `par_map` needs both"`` |

#### Acceptance Criteria (Gherkin)

```gherkin
Feature: cprime-effects

  Scenario: Happy path — concept example passes the effect check (Phase 1 exit criterion)
    Given Auth::generate_token with "uses CryptoCap" and its concept implementation
    When check_effects runs
    Then no diagnostic is returned

  Scenario: Error path — undeclared capability (Phase 1 exit criterion)
    Given a variant of the implementation that also performs Auth::NetCap::send
    And Auth::NetCap is not in generate_token's uses
    When check_effects runs
    Then exactly one UnauthorizedEffect is returned at the node of that perform

  Scenario: Error path — effect inherited through a call
    Given spec g() U64 with "uses Clock" and spec f() U64 with no uses line
    And f's implementation calls Main::g
    When check_effects runs on f
    Then one UnauthorizedEffect names "call to `Main::g` requires `Main::Clock`"

  Scenario: Happy path — mutual recursion closed by the candidate
    Given the accepted Math::odd and candidate Math::even of the Example, both with (decreases n)
    When termination runs on even
    Then component contains even and odd
    And edges contain the edge odd → even with caller_is_candidate false
    And diagnostics are empty

  Scenario: Error path — self recursion without a measure
    Given (impl Math::fact [n] (if (== n 0) 1 (* n (Math::fact (- n 1))))) and fact's uses lacks Diverge
    When termination runs
    Then one MissingDecreases is returned at node 0

  Scenario: Edge case — Diverge exempts recursion
    Given spec serve(n U64) with "uses Diverge" whose implementation calls Main::serve
    When termination runs
    Then diverges is true, diagnostics are empty and edges are empty

  Scenario: Error path — measure length mismatch
    Given accepted Math::odd with (decreases n) and candidate Math::even with (decreases [n n])
    When termination runs on even
    Then a TypeMismatch is returned at even's decreases node

  Scenario: Edge case — non-recursive call chain
    Given f calls g, g calls h, and none calls back
    When termination runs on f
    Then component is empty and edges are empty

  Scenario: Error path — provider signature mismatch
    Given capability CryptoCap { deterministic hmac_sha256(key Bytes, msg FixedBytes[32]) Hash }
    And bind CryptoCap = host::Crypto in the entry module
    When check_bindings runs
    Then a TypeMismatch names parameter 2 and bindings does not contain CryptoCap

  Scenario: Happy path — valid bind
    Given the concept CryptoCap and bind CryptoCap = host::Crypto in the entry module
    When check_bindings runs
    Then bindings maps CryptoCap to provider index 0 and diagnostics are empty

  Scenario: Happy path — parallel map over a thread-safe deterministic capability
    Given Auth::verify_one uses CryptoCap, bound to host::Crypto (thread_safe true, every operation deterministic)
    And an implementation of Batch::check whose body is (par_map Auth::verify_one tokens) and whose spec uses CryptoCap
    When check_effects runs
    Then there are no diagnostics
    And the transitive effect set contains CryptoCap

  Scenario: Error path — mapped spec may diverge
    Given Net::poll_one uses Diverge
    And an implementation body (par_map Net::poll_one sockets)
    When check_effects runs
    Then an InvalidParMap "`Net::poll_one` may diverge; `par_map` needs a terminating spec" is returned

  Scenario: Error path — mapped spec uses a provider that is not thread-safe
    Given Audit::stamp uses ClockCap, bound to host::Clock (thread_safe false)
    And an implementation body (par_map Audit::stamp events)
    When check_effects runs
    Then an InvalidParMap "`Audit::stamp` uses `ClockCap`, which is not deterministic and thread-safe; `par_map` needs both" is returned

  Scenario: Happy path — valid server
    Given the "api.cprime" server example, host::Log thread-safe, and a linked transport "acme::HttpTransport" whose records match Request and Response
    When check_bindings runs
    Then no diagnostic is produced

  Scenario: Error path — handler with a requires clause
    Given the server example where Api::handle has "requires len(req.body) > 0"
    When check_bindings runs
    Then an InvalidServer "handler `Api::handle` has a requires clause; a handler receives untrusted input and cannot have one" is returned

  Scenario: Error path — handler uses a provider that is not thread-safe
    Given the server example where Api::handle uses Store bound to "acme::Redis" with thread_safe false
    When check_bindings runs
    Then an InvalidServer naming "acme::Redis" and "not thread-safe" is returned

  Scenario: Error path — request record mismatch
    Given a transport whose request record has a field "path Bytes" in position 2 and Request declares "method Bytes" there
    When check_bindings runs
    Then an InvalidServer "handler `Api::handle` request type `Api::Request` does not match transport `acme::HttpTransport`: field 2 is `method Bytes`, expected `path Bytes`" is returned

  Scenario: Error path — entry and server together
    Given a project declaring entry Main::main and server Api
    When check_bindings runs
    Then a TypeMismatch "project declares more than one entry: `Main::main`, `Api::Api`" is returned
```

#### Performance, Security, Observability

- **Performance targets:** `check_effects` ≤ 0.5 ms p95 and `termination` ≤ 1 ms p95 for a candidate of ≤ 200 IR instructions whose reachable call subgraph has ≤ 100 specs, warm database (inside T0 ≤ 5 ms, G5). `check_bindings` is memoised; cold ≤ 5 ms for ≤ 100 `.cprime` files.
- **Security:** this crate is the enforcement point of "no ambient authority" (ADR-005): every effect reaching a submission is attributed either to a `Perform` or to a callee's declared `uses`, and must be in the spec's `uses`. Binding checks prevent a provider whose signature or determinism differs from the capability being bound (ADR-021). Inputs are untrusted; walks are iterative over bounded trees (the AST depth guard of `cprime-types` applies first).
- **Observability:** one `debug` log line per public call (target `cprime_effects`, fields `fn`, `spec`, `diagnostics`, `duration_us`, plus `component_size` for `termination`). No metrics or spans.

#### Gaps

None.
