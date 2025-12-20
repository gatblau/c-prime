# Component spec — cprime-types

> **Spec set:** cprime
> **Status:** Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> **Last updated:** 2025-12-20

### SPEC: cprime-types
**File:** `crates/cprime-types/src/lib.rs` (modules `resolve_ty.rs`, `mono.rs`, `spec.rs`, `contract.rs`, `complete.rs`, `check.rs`, `lower.rs`, `deps.rs`, `diag.rs`) | **Package:** `cprime-types` | **Phase:** 1 (checking), 2 (lowering) | **Dependencies:** `cprime-core` (workspace); `salsa`, `blake3` (third-party); dev-dependency `cprime-syntax`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprime-types` turns resolved syntax into typed, monomorphised structures that every later stage trusts. It checks spec declarations (parameter and result types, `uses`, waivers, contract clauses), enforces the result-completeness rules (`UnconstrainedOutput`, ADR-017 and ADR-026), checks an agent's `.opq` implementation against its spec (arity, types, exhaustive `match`), and lowers an accepted-candidate typed implementation to C′-IR before reference counting. It is the T0 type-checking stage and the first T1 stage of the pipeline.

#### Approach

The checker is bidirectional: `check(expr, expected)` pushes a known type into literals, empty vectors and generic constructors; `infer(expr)` synthesises a type everywhere else. Generic type definitions are monomorphised eagerly while types are resolved, so every `Type` leaving the crate names a concrete instance `DefId` and the IR has no type parameters. Internally the checker uses its own `Ty` enum with an `Error` variant so one mistake does not cascade into many diagnostics; `Ty` is converted to the shared `Type` only when no diagnostic was produced. Lowering is a single recursive pass over the typed AST that appends instructions to the current block and creates join blocks for `If`, `And`, `Or` and `Match`. Rejected alternatives: Hindley–Milner inference with unification variables everywhere (agents never write type annotations inside bodies, but every binding's type is fixed by its initialiser, so local inference suffices and keeps diagnostics local); lazy monomorphisation at IR time (the typed AST would then carry type parameters into `cprime-effects` and `cprime-verify`).

#### Shared Context

Copied from `phase-2-architecture.md` §2C and §2E. Only the items this crate uses.

```rust
// cprime_core::ids
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct Symbol(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct DefId(pub [u8; 32]);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct CapId(pub u32);   // CapId(0) = Diverge
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, serde::Serialize)] #[serde(transparent)] pub struct NodeId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct ValueId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct BlockId(pub u32);

// cprime_core::ast (consumed)
pub struct Ident { pub name: String, pub span: Span }
pub struct Path { pub segments: Vec<Ident>, pub span: Span }
pub struct Module { pub file: String, pub path: Path, pub items: Vec<Item> }
pub enum Item { Type(TypeDef), Capability(CapabilityDef), Spec(SpecDef), Entry(EntryDef), Bind(BindDef), Server(ServerDef) }
pub struct TypeDef { pub name: Ident, pub generics: Vec<Ident>, pub body: TypeDefBody, pub span: Span }
pub enum TypeDefBody { Alias(TypeExpr), Record(Vec<FieldDecl>), Sum(Vec<VariantDecl>) }
pub struct FieldDecl { pub name: Ident, pub ty: TypeExpr }
pub struct VariantDecl { pub name: Ident, pub fields: Vec<FieldDecl>, pub span: Span }
pub struct TypeExpr { pub path: Path, pub args: Vec<TypeArg>, pub span: Span }
pub enum TypeArg { Type(TypeExpr), Nat(u32, Span) }
pub struct CapabilityDef { pub name: Ident, pub ops: Vec<CapOpDecl>, pub span: Span }
pub struct CapOpDecl { pub deterministic: bool, pub name: Ident, pub params: Vec<Param>, pub ret: TypeExpr, pub span: Span }
pub struct Param { pub name: Ident, pub ty: TypeExpr }
pub struct SpecDef {
    pub name: Ident, pub params: Vec<Param>, pub ret: TypeExpr, pub uses: Vec<Path>,
    pub assurance: Option<(AssuranceLit, Span)>, pub unconstrained: Vec<WaiverItem>,
    pub requires: Vec<ContractExpr>, pub ensures: Option<(Ident, Vec<ContractExpr>)>, pub span: Span,
}
pub enum AssuranceLit { Proved, Tested }
pub enum WaiverItem { Field(Ident), VariantField(Ident, Ident), VariantTest(Span) }
pub struct EntryDef { pub name: Ident, pub params: Vec<Param>, pub ret: TypeExpr, pub uses: Vec<Path>, pub span: Span }
pub struct ContractExpr { pub span: Span, pub text: String, pub kind: ContractExprKind }
pub enum ContractExprKind {
    Int(i128), Bool(bool), Bytes(Vec<u8>), Unit, Path(Path),
    Call { callee: Path, args: Vec<ContractExpr> },
    Field { value: Box<ContractExpr>, name: Ident },
    Is { value: Box<ContractExpr>, variant: Ident },
    Unary { op: UnaryOp, operand: Box<ContractExpr> },
    Binary { op: BinaryOp, lhs: Box<ContractExpr>, rhs: Box<ContractExpr> },
}
pub enum UnaryOp { Not, Neg }
pub enum BinaryOp { Implies, Or, And, Eq, Ne, Lt, Le, Gt, Ge, Concat, Add, Sub, Mul, Div, Rem }
pub struct Impl { pub file: String, pub header: Option<String>, pub spec: Path, pub params: Vec<(Ident, NodeId)>,
                  pub decreases: Option<Decreases>, pub body: Expr, pub node_count: u32 }
pub struct Decreases { pub node: NodeId, pub measure: Vec<Expr> }
pub struct Expr { pub node: NodeId, pub span: Span, pub kind: ExprKind }
pub enum ExprKind {
    Int(i128), Bool(bool), Bytes(Vec<u8>), Unit, Name(Path),
    Let { bindings: Vec<(Ident, NodeId, Expr)>, body: Box<Expr> },
    If { cond: Box<Expr>, then_: Box<Expr>, else_: Box<Expr> },
    Match { scrutinee: Box<Expr>, arms: Vec<Arm> },
    Perform { op: Path, args: Vec<Expr> },
    Apply { head: Path, args: Vec<Expr> },
    Field { value: Box<Expr>, name: Ident },
    Vec { items: Vec<Expr> },
    ParMap { spec: Path, xs: Box<Expr> },
}
pub struct Arm { pub node: NodeId, pub variant: Path, pub binders: Vec<(Ident, NodeId)>, pub body: Expr }

// cprime_core::ir (produced by lower)
pub enum Type { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<Type>), Struct(DefId), Sum(DefId),
                Fn { params: Vec<Type>, ret: Box<Type>, effects: BTreeSet<CapId> }, Token }
pub enum Const { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(Vec<u8>) }
pub enum BinOp { Add, Sub, Mul, Div, Rem, WrappingAdd, WrappingSub, WrappingMul, Eq, Ne, Lt, Le, Gt, Ge, And, Or, BytesConcat }
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
    RcDup(ValueId), RcDrop(ValueId), DropReuse(ValueId),          // never emitted by this crate
}
pub enum TrapKind { Overflow, DivisionByZero, IndexOutOfBounds, Unreachable }
pub enum Terminator {
    Return(ValueId), Jump { target: BlockId, args: Vec<ValueId> },
    Branch { cond: ValueId, then_target: BlockId, then_args: Vec<ValueId>, else_target: BlockId, else_args: Vec<ValueId> },
    Switch { scrutinee: ValueId, cases: Vec<(BlockId, Vec<ValueId>)> }, Trap(TrapKind),
}
pub struct Block { pub params: Vec<(ValueId, Type)>, pub insts: Vec<(ValueId, Inst)>, pub term: Terminator }
pub struct Function { pub def: DefId, pub ty: Type, pub entry: BlockId, pub blocks: Vec<Block>, pub value_count: u32,
                     pub node_map: BTreeMap<ValueId, NodeId> }  // (D3, D23) source node of every parameter, non-constant instruction, join parameter and Return value
pub struct Layouts { pub structs: BTreeMap<DefId, Vec<Type>>, pub sums: BTreeMap<DefId, Vec<Vec<Type>>> }

// cprime_core::tast (produced) — TSpec, Waiver, TImpl, TDecreases, TExpr, TExprKind, TArm, TClause, CExpr, CExprKind, CBinOp
pub struct TSpec { pub def: DefId, pub name: String, pub params: Vec<(Symbol, Type)>, pub ret: Type,
                   pub uses: BTreeSet<CapId>, pub assurance: Assurance, pub waivers: Vec<Waiver>,
                   pub requires: Vec<TClause>, pub ensures: Vec<TClause>, pub is_entry: bool }
pub enum Waiver { Field(Symbol), VariantField { variant: Symbol, field: Symbol }, VariantTest }
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
pub struct TClause { pub text: String, pub span: Span, pub expr: CExpr }
pub struct CExpr { pub ty: Type, pub kind: CExprKind }
pub enum CExprKind {
    Int(i128), Const(Const), Param(u32), Result,
    Bin { op: CBinOp, lhs: Box<CExpr>, rhs: Box<CExpr> }, Not(Box<CExpr>),
    And(Box<CExpr>, Box<CExpr>), Or(Box<CExpr>, Box<CExpr>), Implies(Box<CExpr>, Box<CExpr>),
    Is { value: Box<CExpr>, tag: u32 }, Field { value: Box<CExpr>, index: u32 },
    VariantField { value: Box<CExpr>, tag: u32, index: u32 },
    Intrinsic { op: IntrinsicOp, args: Vec<CExpr> }, Len(Box<CExpr>), Index { vec: Box<CExpr>, index: Box<CExpr> },
    OpApp { cap: CapId, op: Symbol, qualified: String, args: Vec<CExpr> },
}
pub enum CBinOp { Add, Sub, Mul, Div, Rem, Eq, Ne, Lt, Le, Gt, Ge, Concat }

// cprime_core::evidence
pub enum Assurance { Proved, Tested }

// cprime_core::value
pub struct TypeSchema {
    pub structs: BTreeMap<DefId, Vec<(String, Type)>>,
    pub sums: BTreeMap<DefId, Vec<(String, Vec<(String, Type)>)>>,
}

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

// cprime_core::db, cprime_core::resolve, cprime_core::namespace (called)
pub trait Db: salsa::Database {
    fn inputs(&self) -> ProjectInputs; fn interner(&self) -> &Interner;
    fn parse_module(&self, file: SourceFile) -> Arc<ParsedModule>;
    fn accepted_impl(&self, spec: DefId) -> Option<Arc<AcceptedImpl>>;
    fn impl_status(&self, spec: DefId) -> ImplStatus;
}
pub struct NamespaceTable { pub entries: BTreeMap<String, NsEntry> }
pub struct NsEntry { pub def: DefId, pub kind: NsKind, pub file: String, pub span: Span }
pub enum NsKind { Type, Capability, Spec, Entry, Server, Variant { sum: DefId, tag: u32 } }
pub enum DefKind { Type, Capability, Spec, Entry, Prelude }
pub struct Scope<'a> { pub module: &'a str, pub locals: &'a [Symbol] }
pub enum Res { Local(u32), Def { def: DefId, kind: DefKind }, Variant { sum: DefId, tag: u32 },
               CapOp { cap: CapId, cap_def: DefId, op: Symbol }, Prelude(PreludeItem), Builtin(BuiltinType) }
pub enum PreludeItem { Hash, Result, Option, Len, Index, Concat, U64Be, Push, Slice, ToBytes, U8Max, U64Max, I64Min, I64Max, Diverge }
pub enum BuiltinType { Unit, Bool, U8, U64, I64, Bytes, FixedBytes, Vec }
pub fn namespace_table(db: &dyn Db) -> Arc<NamespaceTable>;
pub fn lookup(db: &dyn Db, scope: &Scope, path: &Path) -> Result<Res, Diagnostic>;   // emits UnknownName on failure
```

Prelude members (§2C): `Hash = FixedBytes[32]`; `enum Result[T, E] { Ok { value T }; Err { error E } }` (tags `Ok`=0, `Err`=1); `enum Option[T] { Some { value T }; None }` (tags `Some`=0, `None`=1); `len`, `index`, `++`, `u64_be`, `push`, `slice`, `to_bytes`; constants `U8::MAX`, `U64::MAX`, `I64::MIN`, `I64::MAX`; capability `Diverge` = `CapId(0)`.

Layout limits (§2C heap layout): at most 64 fields per struct or variant; at most 65,536 variants per sum.

Runtime protocol 2 (§2E): trap sites are implicit in `Bin` `Add`/`Sub`/`Mul`/`Div`/`Rem`, `Index` and `Intrinsic Slice`; lowering emits no explicit check branches.

**Parallel map (RFC 02, D14, D15).** `.opq` `(par_map S xs)` parses to `ExprKind::ParMap { spec, xs }`; this crate types it as `TExprKind::ParMap { func, xs }` of type `Vec(ret(func))` and lowers it to `Inst::ParMap { func, xs }`. Checks, in order: `spec` resolves to a spec (else `InvalidParMap`); the spec has one parameter (else `ArityMismatch`); `xs : Vec[T]` with `T` equal to the parameter type (else `TypeMismatch`); the spec has no `requires` (else `InvalidParMap`). The effect restrictions on the mapped spec (D16) belong to `cprime-effects`.

#### Public Interface

All functions are free functions in the crate root. `check_spec` and `capability_ops` are Salsa-tracked (memoised per `DefId`/`CapId` and invalidated when their inputs change); the others are plain functions.

```rust
pub struct SpecCheck { pub spec: TSpec, pub schema: TypeSchema, pub diagnostics: Vec<Diagnostic> }
pub struct ImplCheck { pub timpl: Option<TImpl>, pub diagnostics: Vec<Diagnostic> }
pub struct CapOpSig { pub name: Symbol, pub params: Vec<Type>, pub ret: Type, pub deterministic: bool }
// deterministic (ADR-054): equal results for equal arguments, no observable effect, returns without waiting on anything outside the call

/// Checks a `spec` or `entry` declaration. `def` must be the DefId of a namespace entry of kind Spec or Entry.
/// Diagnostics empty ⇔ the spec is usable. When diagnostics are non-empty, `spec` is still populated:
/// unresolvable types become `Type::Unit` and ill-typed clauses are omitted.
pub fn check_spec(db: &dyn Db, def: DefId) -> Arc<SpecCheck>;

/// Checks a submitted implementation against a clean spec. Precondition: `check_spec(spec.def).diagnostics` is empty
/// (cprime-daemon guarantees it). `timpl` is Some ⇔ `diagnostics` is empty.
pub fn check_impl(db: &dyn Db, spec: &TSpec, imp: &ast::Impl) -> ImplCheck;

/// Operations of a capability in declaration order. CapId(0) (Diverge) returns an empty vector.
pub fn capability_ops(db: &dyn Db, cap: CapId) -> Arc<Vec<CapOpSig>>;

/// Lowers a typed implementation to C′-IR before RC insertion and fills `node_map` (D3). Never fails; does not run
/// the IR verifier: cprime-daemon and cprime-cli run `ir_verify::verify_function(.., IrStage::Lowered)` on the result and
/// treat failure as a daemon failure (A-17); this crate calls the verifier only from its tests (§2E).
pub fn lower(db: &dyn Db, spec: &TSpec, imp: &TImpl) -> (ir::Function, Layouts);

/// DefIds whose change invalidates an accepted implementation's verification (input to the context hash).
pub fn verification_deps(db: &dyn Db, spec: &TSpec, imp: &TImpl) -> BTreeSet<DefId>;
```

Cross-crate helper (§2E, D11; called by `cprime-effects` to recognise prelude `Result` instances):

```rust
pub fn mono_instance_id(generic: DefId, args: &[Type]) -> DefId;
```

##### Example

Input `.cprime` (`auth.cprime`) and `.opq` from concept §4.1–4.2:

```text
module Auth

capability CryptoCap {
    deterministic hmac_sha256(key, msg Bytes) Hash
}

struct SessionToken {
    user_id    U64
    expires_at U64
    sig        Hash
}

spec generate_token(user_id, now U64, secret Bytes) SessionToken {
    uses CryptoCap
    requires user_id > 0
    requires now <= U64::MAX - 3600
    requires len(secret) >= 32
    ensures result.user_id == user_id
    ensures result.expires_at == now + 3600
    ensures result.sig == CryptoCap::hmac_sha256(secret, u64_be(user_id) ++ u64_be(result.expires_at))
}
```
```lisp
(impl Auth::generate_token [uid now secret]
  (let [exp (+ now 3600)
        msg (++ (u64_be uid) (u64_be exp))
        sig (perform CryptoCap::hmac_sha256 secret msg)]
    (SessionToken uid exp sig)))
```

`check_spec` returns `diagnostics: []`, `spec.ret = Type::Struct(<SessionToken DefId>)`, `spec.uses = {CapId(1)}`, three `requires` and three `ensures` clauses; the completeness check finds `user_id`, `expires_at` and `sig` mentioned. `check_impl` returns a `TImpl` whose body is `Let(Bin Add(Local 1, Const U64(3600)), Let(Bin BytesConcat(Intrinsic U64Be[Local 3], Intrinsic U64Be[Local 0]), Let(Perform hmac_sha256[Local 2, Local 0], MakeStruct[Local 5, Local 2, Local 0])))` — de Bruijn indices count outwards from the innermost binder (parameters `uid`, `now`, `secret` are bound first, `secret` innermost). `lower` returns (value numbers as produced by Internal Logic step L-3):

```text
function Auth::generate_token  ty = Fn([U64, U64, Bytes]) -> Struct(SessionToken) effects {CapId(1)}
block0(v0: U64, v1: U64, v2: Bytes):
  v3 = Const U64(3600)
  v4 = Bin Add v1, v3                 ; trap site: Overflow
  v5 = Intrinsic U64Be [v0]
  v6 = Intrinsic U64Be [v4]
  v7 = Bin BytesConcat v5, v6
  v8 = Perform CapId(1) hmac_sha256 [v2, v7]
  v9 = MakeStruct SessionToken [v0, v4, v8] reuse=None
  Return v9
value_count = 10
node_map = { v0: 1, v1: 2, v2: 3, v4: 6, v5: 11, v6: 13, v7: 10, v8: 16, v9: 19 }   ; parameters uid, now, secret; `(+ now 3600)`; `(u64_be uid)`; `(u64_be exp)`; `(++ …)`; Perform; Return value = `(SessionToken uid exp sig)` (cprime-syntax pre-order ids); the Const v3 has no entry (D23)
layouts.structs = { SessionToken: [U64, U64, FixedBytes(32)] }
```

Removing `result.sig == …` from `ensures` yields:

```json
{"kind":"UnconstrainedOutput","tier":"t0","severity":"error","node":0,
 "span":{"file":"auth.cprime","line":4,"column":1},
 "message":"result field `sig` of `Auth::generate_token` is neither mentioned by an ensures clause nor waived in unconstrained",
 "clause":"result.sig"}
```

#### Internal Logic

Common to every step: diagnostics carry `tier: T0`, `severity: Error`. Diagnostics about `.cprime` declarations carry `node: NodeId(0)` and a `span` into the `.cprime` file (SA-32). Diagnostics about the submission carry the offending AST node's `node` and `span` (file `"<submission>"` as given by `cprime-syntax`). Each public function logs one `debug` line on exit with target `cprime_types`, fields `fn`, `spec` (qualified name), `diagnostics` (count), `duration_us`. No source text is logged.

**T — type resolution and monomorphisation (used by every function)**

T-1. `resolve_type(scope, TypeExpr) -> Ty` calls `cprime_core::resolve::lookup` on `TypeExpr.path` (scope = module of the declaration, no locals). `lookup` failure → its `UnknownName` diagnostic is recorded and `Ty::Error` returned.
T-2. `Res::Builtin(b)`: `Unit`, `Bool`, `U8`, `U64`, `I64`, `Bytes` require zero args; `FixedBytes` requires exactly one `TypeArg::Nat(n)` with `1 ≤ n ≤ 65,536`; `Vec` requires exactly one `TypeArg::Type`. Any other argument shape → `TypeMismatch` "`<path>` expects <k> type argument(s), found <m>" or "FixedBytes length must be between 1 and 65536, found <n>".
T-3. `Res::Prelude(Hash)` → `FixedBytes(32)`. `Res::Prelude(Result|Option)` and `Res::Def{kind: Type}` → look up the `TypeDef` (prelude definitions are built into `cprime-core`); argument count must equal `generics.len()` and every argument must be `TypeArg::Type`, else `TypeMismatch`. Any other `Res` → `TypeMismatch` "`<path>` is not a type".
T-4. Alias bodies are expanded by substituting arguments for generics and resolving the alias body in the alias's own module scope. An alias chain that revisits a definition → `TypeMismatch` "type alias cycle through `<name>`".
T-5. Record and sum definitions produce an instance: `mono_instance_id(generic, args)`. With zero generics the instance id is the definition's own `DefId`. Otherwise it is `blake3("cprime.mono.v1\0" ‖ generic.0 ‖ enc(arg_1) ‖ … ‖ enc(arg_k))` where `enc` is: `Unit` 0x00, `Bool` 0x01, `U8` 0x02, `U64` 0x03, `I64` 0x04, `Bytes` 0x05, `FixedBytes(n)` 0x06 ‖ n as u32 little-endian, `Vec(t)` 0x07 ‖ enc(t), `Struct(d)` 0x08 ‖ d.0, `Sum(d)` 0x09 ‖ d.0. The instance is recorded in a per-query `MonoTable { instances: BTreeMap<DefId, (DefId /* generic */, Vec<Type>)> }`, and its fields (or variants) are resolved with generics substituted. Instantiation depth (instances created while resolving one instance's fields) above 32 → `TypeMismatch` "monomorphisation of `<name>` exceeds depth 32".
T-6. Layout limits: a record or variant with more than 64 fields → `TypeMismatch` "`<name>` has <n> fields; the limit is 64"; a sum with more than 65,536 variants → `TypeMismatch` "`<name>` has <n> variants; the limit is 65536".
T-7. Finite inhabitation (ADR-023): after all instances reachable from a declaration are resolved, compute the least fixed point `inhabited`: scalars, `Unit`, `Bytes`, `FixedBytes`, `Vec` are inhabited; a struct is inhabited when every field type is; a sum is inhabited when at least one variant has all field types inhabited. Any struct or sum instance not inhabited → `TypeMismatch` "type `<name>` has no finite values". Example rejected: `struct T { t T }`. Example accepted: `enum List { Nil; Cons { head U64; tail List } }`.
T-8. `TypeSchema` is filled for every struct and sum instance reachable from the spec signature: field names are the declaration's identifier text, variant index = declaration order = tag.

**S — `check_spec(db, def)`**

S-1. Find the namespace entry whose `def` equals `def` via `namespace_table`; read its file with `Db::parse_module` and locate the `SpecDef` or `EntryDef` by name. No such entry, or kind not Spec/Entry → return a `SpecCheck` with `spec.name = "<unknown>"`, empty fields and one `UnknownName` "no spec or entry with this identity". (Daemon only calls with ids from the namespace table, so this row exists for defensive completeness.)
S-2. Resolve each parameter type and the result type with T-1…T-8. Duplicate parameter names → `TypeMismatch` "parameter `<p>` is declared twice".
S-3. Resolve each `uses` path: `Res::Def{kind: Capability}` → `CapId` interned from the capability's qualified name; `Res::Prelude(Diverge)` → `CapId(0)`; anything else → `TypeMismatch` "`<path>` in uses is not a capability". Duplicates are merged without a diagnostic.
S-4. For each capability in `uses` other than `Diverge`, call `capability_ops` and append the capability's diagnostics (step C-3) once per spec.
S-5. Assurance: absent → `Proved`. Entries: `assurance` is always `Proved`, `requires`/`ensures` empty, `is_entry = true` (SA-29).
S-6. Waivers (only for specs): the result type decides validity. `Field(f)`: result is a struct with field `f`. `VariantField(V, f)`: result is a sum with variant `V` that has field `f`. `VariantTest`: result is a sum. Otherwise `TypeMismatch` "waiver `<item>` does not name a part of the result type `<ty>`". A waiver listed twice → `TypeMismatch` "waiver `<item>` is listed twice".
S-7. Type each `requires` clause with K-1…K-9 in the parameter environment, then each `ensures` clause in the parameter environment plus the binder. The binder name equal to a parameter name → `TypeMismatch` "ensures binder `<b>` shadows parameter `<b>`". A clause whose type is not `Bool` → `TypeMismatch` "contract clause has type `<ty>`, expected `Bool`". A clause with any diagnostic is omitted from the `TSpec`.
S-8. Run the completeness check (step P) on specs (not entries) with a struct or sum result.
S-9. Convert `Ty` to `Type`; build `TSpec` (`name` = qualified, e.g. `Auth::generate_token`; `params` interned names), `TypeSchema`, return `Arc<SpecCheck>`.

**K — contract expression typing (SA-19)**

K-1. Literals: `Int(n)` → `CExprKind::Int(n)`; its type is the type of the other operand of the enclosing binary operator when that operand is typed first, else `U64`. Contracts use mathematical integers, so no range check is applied. `Bool`, `Unit` → `Const`. `Bytes(b)` → `Const(Bytes)` of type `Bytes`, or `FixedBytes(n)` when compared against a `FixedBytes(n)` operand and `b.len() == n`.
K-2. `Path` with one segment equal to a parameter name → `Param(i)`; equal to the ensures binder (ensures only) → `Result`. Otherwise `lookup`: `Prelude(U8Max|U64Max|I64Min|I64Max)` → `Int(255 | 2^64−1 | −2^63 | 2^63−1)` typed `U8`/`U64`/`I64`/`I64`. The binder used inside `requires` → `UnknownName` (lookup failure). Any other resolution → `TypeMismatch` "`<path>` is not a value in a contract".
K-3. `Field{value, name}`: when `value` is itself `Field{value: e, name: V}` and `e` has a sum type with variant `V`, the pair is `VariantField{e, tag(V), index(name)}`. Otherwise `value` must have a struct type with field `name` → `Field`. A bare variant projection `e.V` → `TypeMismatch` "variant projection `e.V` must be followed by a field name". Unknown field → `TypeMismatch` "type `<ty>` has no field `<f>`".
K-4. `Is{value, variant}`: `value` must have a sum type containing `variant` → `Is{tag}` of type `Bool`; else `TypeMismatch`.
K-5. `Unary Not` needs `Bool`; `Unary Neg` needs an integer type and keeps it.
K-6. `Binary`: `Implies`, `Or`, `And` need `Bool` operands; `Add`, `Sub`, `Mul`, `Div`, `Rem` need equal integer types and yield that type; `Lt`, `Le`, `Gt`, `Ge` need equal integer types and yield `Bool`; `Eq`, `Ne` need equal types (any type except `Fn`, `Token`) and yield `Bool`; `Concat` needs `Bytes` and `Bytes`. Mismatch → `TypeMismatch` "operator `<op>` cannot combine `<lhs>` and `<rhs>`".
K-7. `Call{callee, args}` resolves `callee`: `Prelude(Len|Index|U64Be|Push|Slice|ToBytes)` → `Len`, `Index` or `Intrinsic` with argument rules of step E-8; `CapOp` whose op is `deterministic` → `OpApp` with `qualified` = `"<Module>::<Capability>::<op>"`; a non-deterministic op → `TypeMismatch` "operation `<q>` is not deterministic and cannot appear in a contract"; a spec or entry → `TypeMismatch` "specs cannot be called from contracts"; a type or variant → `TypeMismatch` "contracts cannot construct values". Argument count mismatch → `TypeMismatch` "`<callee>` takes <k> argument(s), found <m>".
K-8. A `CapOp` used in a contract must belong to a capability in the spec's `uses`; otherwise `TypeMismatch` "contract uses `<q>` but `<Cap>` is not in the uses list".
K-9. Every `CExpr` records its `Type`. The `TClause.text` is `ContractExpr.text` verbatim; `span` is its span.

**P — completeness (ADR-017, ADR-026, SA-13)**

P-1. Walk every `ensures` clause's `CExpr` tree. Record `Field{value: Result, index: i}` as "field i mentioned"; record `Is{value: Result, tag}` as "variant test present"; record `VariantField{value: Result, tag, index}` as "variant field (tag, index) mentioned". A mention inside a larger path (for example `result.inner.x`) counts, because the subtree contains `Field{Result, inner}`.
P-2. Struct result: for each field not mentioned and not waived by `Waiver::Field` → one `UnconstrainedOutput`, `clause: "<binder>.<field>"`, message "result field `<field>` of `<spec>` is neither mentioned by an ensures clause nor waived in unconstrained".
P-3. Sum result: no variant test and no `VariantTest` waiver → `UnconstrainedOutput`, `clause: "<binder> is <first variant>"`, message "no ensures clause of `<spec>` tests which variant `<binder>` is". For every variant `V` and field `f` not mentioned and not waived by `VariantField(V, f)` → `UnconstrainedOutput`, `clause: "<binder>.<V>.<f>"`, message "field `<f>` of variant `<V>` of the result of `<spec>` is neither mentioned nor waived". Field-less variants need only the variant test.
P-4. Any other result type (scalars, `Unit`, `Bytes`, `FixedBytes`, `Vec`) has no completeness requirement; a spec without `ensures` and with such a result passes P.
P-5. Diagnostics are ordered: field and variant order of the result type.

**C — `capability_ops(db, cap)`**

C-1. `CapId(0)` → empty vector. Otherwise map `CapId` → qualified name through the interner, find the `CapabilityDef` through `namespace_table` and `Db::parse_module`.
C-2. Resolve each op's parameter and result types (T-1…T-6) in the capability's module.
C-3. Diagnostics (returned to `check_spec` callers via S-4, held in a crate-internal tracked companion query `capability_check(db, cap) -> Arc<Vec<Diagnostic>>`): two ops with the same name → `TypeMismatch` "capability `<Cap>` declares operation `<op>` twice"; a parameter or result type that is not a provider boundary type (`Unit`, `Bool`, `U8`, `U64`, `I64`, `Bytes`, `FixedBytes[N]`, `Vec[B]`, `Result[B1, B2]`, where every `B` is itself a boundary type) → `TypeMismatch` "type `<ty>` of operation `<op>` cannot cross the provider boundary". Ops with diagnostics are omitted from the returned vector.

**E — `check_impl(db, spec, imp)`**

E-1. Resolve `imp.spec` with `lookup(Scope{module: <spec's module>, locals: []})`; it must be `Res::Def{def == spec.def}`, else `TypeMismatch` at node 0 "implementation names `<path>` but was submitted for `<spec>`".
E-2. Arity: `imp.params.len() != spec.params.len()` → `ArityMismatch` at node 0 "`<spec>` takes <k> parameter(s); the implementation binds <m>" and return immediately (`timpl: None`). Two binders with the same name → `TypeMismatch` at the second binder's node.
E-3. Initialise the local environment with the parameters (parameter 0 outermost). Locals are a stack of `(Symbol, Ty)`; `lookup` receives the symbol list so that `Res::Local(i)` indexes from the innermost.
E-4. Nesting guard: expression depth above 256 → `TypeMismatch` "expression nesting exceeds 256" at the first node beyond the limit; the subtree is not visited.
E-5. If `imp.decreases` is present, check every measure element against `U64` (E-6). The measure is checked in the parameter environment only.
E-6. `check(e, expected)` / `infer(e)` per `ExprKind`:
  - `Int(n)`: checked against `U8`/`U64`/`I64` → range check (`0..=255`, `0..=2^64−1`, `−2^63..=2^63−1`); inferred → `U64` when `n ≥ 0`, `I64` when `n < 0`. Out of range → `TypeMismatch` "literal <n> does not fit in `<ty>`". `Bool` → `Bool`; `Unit` → `Unit`; `Bytes(b)` → `Bytes`, or `FixedBytes(n)` when checked against `FixedBytes(n)` and `b.len() == n`.
  - `Name(path)`: `Local(i)` → its type; `Prelude(U8Max|U64Max|I64Min|I64Max)` → typed `Const`; `Variant{sum, tag}` with zero fields → `MakeVariant` (generic arguments from the expected type, else `TypeMismatch` "cannot infer the type arguments of `<V>`"); a variant with fields → `TypeMismatch` "variant `<V>` takes <k> field(s)"; a spec or entry → `TypeMismatch` "`<path>` is a function and cannot be used as a value" (SA-12); anything else → `TypeMismatch` "`<path>` is not a value".
  - `Let{bindings, body}`: each initialiser is inferred in the environment extended by the previous bindings; the body is checked against `expected` (or inferred). Lowered to nested `TExprKind::Let`, one per binding.
  - `If`: condition checked `Bool`; with `expected`, both branches checked against it; without, `then_` inferred and `else_` checked against its type.
  - `Match`: scrutinee inferred, must be a sum instance (`TypeMismatch` "match scrutinee has type `<ty>`, expected a sum type"). Each arm's `variant` path resolves to `Res::Variant{sum, tag}` whose `sum` equals the scrutinee instance's generic definition (`TypeMismatch` "variant `<V>` does not belong to `<ty>`"); binder count equals the variant's field count (`TypeMismatch` "variant `<V>` has <k> field(s); the arm binds <m>"); a variant covered twice → `TypeMismatch` at the second arm; uncovered variants → one `NonExhaustiveMatch` at the match node "match on `<ty>` does not cover <V1>, <V2>". Arms are emitted sorted by tag; binders are pushed in field order; arm bodies follow the `If` rule.
  - `Perform{op, args}`: `op` must resolve to `Res::CapOp`; argument count equals the op's parameter count (`TypeMismatch` "operation `<q>` takes <k> argument(s), found <m>"); each argument checked against the parameter type; result = op result type; `deterministic` copied from `capability_ops`.
  - `Apply{head, args}`: a one-segment head that is an operator symbol uses E-7. Otherwise resolve `head`: `Prelude(Len|Index|U64Be|Push|Slice|ToBytes|Concat)` → E-8; `Def{kind: Spec}` → `Call` (callee `check_spec`; argument count equals parameter count, `TypeMismatch` otherwise; each argument checked; result = callee result type; a callee whose `check_spec` has diagnostics → `TypeMismatch` "callee `<q>` has errors in its declaration"); `Def{kind: Entry}` → `TypeMismatch` "entry points cannot be called"; `Def{kind: Type}` naming a record → `MakeStruct` with positional fields in declaration order; `Variant` with fields → `MakeVariant` positional. Field count mismatch → `TypeMismatch` "`<T>` has <k> field(s); <m> given". Generic arguments of a constructor come from the expected type when it is an instance of the same definition; otherwise from first-order matching of field types against inferred argument types, left to right; an unresolved type parameter → `TypeMismatch` "cannot infer the type arguments of `<T>`".
  - `Field{value, name}`: `value` inferred; must be a struct instance with field `name` → `GetField{index}`; else `TypeMismatch`.
  - `ParMap{spec, xs}` (D15): resolve `spec`; not `Def{kind: Spec}` → `InvalidParMap` "`par_map` needs a spec; `<path>` is a <kind>" (kind: `capability operation`, `entry`, `type`, `variant`, `local`, `constant`). `check_spec(spec)`; parameter count ≠ 1 → `ArityMismatch` "`<S>` takes <k> parameter(s); `par_map` passes 1". `xs` inferred; not `Vec(t)` with `t` equal to the parameter type → `TypeMismatch` "`par_map` over `<type of xs>` needs `Vec[<T>]` for `<S>`". Non-empty `requires` → `InvalidParMap` "`<S>` has a requires clause; `par_map` needs a spec without one". Result `Vec(ret(S))` → `TExprKind::ParMap{func: S, xs}`.
  - `Vec{items}`: element type from `expected` `Vec[T]` when present, else inferred from the first item; remaining items checked against it. Empty without expectation → `TypeMismatch` "cannot infer the element type of an empty vec".
E-7. Operator heads (exactly the listed arity, else `TypeMismatch` "operator `<op>` takes <k> operand(s), found <m>"):

| Head | Operands | Result | TAST |
|---|---|---|---|
| `+ - * / %` | 2, same integer type | that type | `Bin Add/Sub/Mul/Div/Rem` |
| `wrapping_add wrapping_sub wrapping_mul` | 2, same integer type | that type | `Bin WrappingAdd/…` |
| `< <= > >=` | 2, same integer type | `Bool` | `Bin Lt/Le/Gt/Ge` |
| `== !=` | 2, same type, not `Fn`/`Token` | `Bool` | `Bin Eq/Ne` |
| `and` `or` | 2, `Bool` | `Bool` | `And` / `Or` (short-circuit) |
| `not` | 1, `Bool` | `Bool` | `Not` |
| `++` | 2, `Bytes` | `Bytes` | `Bin BytesConcat` |

For two-operand integer operators the operand that is not a literal is inferred first and the literal is checked against its type; two literals → `U64`.
E-8. Prelude function rules (same in contracts, K-7): `len(x)`: `x` is `Bytes`, `FixedBytes(n)` or `Vec(t)` → `U64` (`Len`). `index(v, i)`: `i` checked `U64`; `v` `Vec(t)` → `t`, `Bytes`/`FixedBytes(n)` → `U8` (`Index`). `u64_be(x)`: `U64` → `Bytes`. `push(v, x)`: `v` `Vec(t)`, `x` checked `t` → `Vec(t)`. `slice(b, s, e)`: `Bytes`, `U64`, `U64` → `Bytes`. `to_bytes(f)`: `FixedBytes(n)` → `Bytes`. `++` as a prelude path behaves as the operator.
E-9. The body is checked against `spec.ret`.
E-10. Assign `Type`s, de Bruijn indices (0 = innermost) and copy `NodeId`s from the AST. When any diagnostic was produced, return `timpl: None` and all diagnostics in node order; otherwise `Some(TImpl)`.

**L — `lower(db, spec, imp)`**

L-1. Create `block0` with one parameter per spec parameter (`ValueId(0)…ValueId(n−1)`, types from `spec.params`). The environment maps de Bruijn locals to `ValueId`s.
L-2. `lower_expr(e) -> ValueId` appends to the current block and may change the current block. Operands are lowered left to right; `Let` lowers its value then its body; values defined in a dominating block are used directly (no extra block arguments).
L-3. Mapping: `Const` → `Inst::Const`; `Local` → environment; `Bin` → `Inst::Bin` (no explicit trap checks, runtime protocol 2); `Not` → `Inst::Not`; `Call`, `Perform`, `Intrinsic`, `Len`, `Index`, `GetField`, `MakeVec`, `ParMap` → the instruction of the same name; `MakeStruct`, `MakeVariant` → with `reuse: None`. `ValueId`s are allocated in creation order; block ids in creation order.
L-4. `If`: lower `cond`; create `then`, `else`, `join` blocks (in that order); `join` has one parameter of the expression's type; terminate the current block with `Branch{cond, then [], else []}`; lower each branch in its block and end it with `Jump{join, [v]}`; continue in `join` with its parameter as the result.
L-5. `And(a, b)`: lower `a`; emit `Const Bool(false)` as `f`; create `rhs` and `join(Bool)`; `Branch{a, rhs [], join [f]}`; in `rhs` lower `b` and `Jump{join, [b]}`. `Or` mirrors with `Const Bool(true)` and the targets swapped.
L-6. `Match`: lower the scrutinee; create one case block per tag (no parameters) then `join`; terminate with `Switch{scrutinee, cases: [(case_i, [])]}`; in each case block emit `GetVariantField{scrutinee, tag, index}` for each binder in field order, extend the environment, lower the body, `Jump{join, [v]}`.
L-7. Terminate the final block with `Return(v_body)`. `TDecreases` is not lowered (it feeds `cprime-verify` only).
L-7a. `node_map` (D3, amended by D23): insert `ValueId → NodeId` for (i) every entry-block parameter — the parameter's node from `TImpl.params`; (ii) every instruction emitted for a `TExpr` other than `Const` — that expression's node (`Bin`, `Not`, `Call`, `Perform`, `Intrinsic`, `Len`, `Index`, `GetField`, `MakeStruct`, `MakeVariant`, `GetVariantField` for an arm binder — the `Match` expression's node, `MakeVec`, `ParMap`); (iii) the parameter of every join block created by L-4, L-5 or L-6 — the node of the `If`, `And`, `Or` or `Match` expression; and (iv) for the `Return` terminator, `v_body → NodeId` of the body expression whose value is returned (for a `Let` chain, the innermost body), unless (ii) or (iii) already inserted an entry for `v_body`, which is kept. `Const` instructions — including the `Const Bool` that L-5 emits — get no entry. Every entry the D3 rule produced before D23 keeps its value. Note (D13 consequence): an `Unreachable` trap-site diagnostic takes the node of its block's first instruction when that instruction has an entry; under D23 more first instructions have one, so such diagnostics carry a non-zero node more often. The D13 rule itself is unchanged.
L-8. `Function.def = spec.def`; `ty = Fn{params, ret, effects: spec.uses}`; `entry = BlockId(0)`; `value_count` = next unused id. `Layouts` contains every struct and sum instance reachable from the parameter, result and expression types (field types from the spec's `TypeSchema` extended by the implementation's own instances).

**D — `verification_deps(db, spec, imp)`**

D-1. Return `{spec.def}` ∪ the `DefId` of every spec called or mapped with `par_map` by `imp` ∪ the `DefId` of every capability whose ops `imp` performs (excluding `Diverge`, which has no definition) ∪ the generic definition `DefId` of every struct or sum instance appearing in `imp`'s expression types. Changes reachable through a callee's or a type's own references are covered transitively because `DefId`s hash their references.

#### Data Model

N/A — no persistent storage. In-memory structures: `MonoTable { instances: BTreeMap<DefId, (DefId, Vec<Type>)> }` per query; checker environment `Vec<(Symbol, Ty)>`; internal `enum Ty { …Type variants…, Error }`. Salsa memoises `check_spec`, `capability_ops` and `capability_check` per `cprimed` lifetime; nothing outlives the daemon (A-7).

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Path in a type, contract or body does not resolve | rejected (T0) | `UnknownName` | `{"kind":"UnknownName","message":"name `Auth::Sesion` is not defined", …}` (message from `lookup`) |
| Implementation binds a different number of parameters | rejected (T0) | `ArityMismatch` | `"message":"`Auth::generate_token` takes 3 parameter(s); the implementation binds 2"` |
| Implementation names a different spec | rejected (T0) | `TypeMismatch` | `"message":"implementation names `Auth::other` but was submitted for `Auth::generate_token`"` |
| Wrong type, operator operands, argument count, field, literal range | rejected (T0) | `TypeMismatch` | `"message":"operator `+` cannot combine `U64` and `I64`"` |
| Type argument shape wrong / FixedBytes length out of range | rejected (T0) | `TypeMismatch` | `"message":"FixedBytes length must be between 1 and 65536, found 0"` |
| Alias cycle | rejected (T0) | `TypeMismatch` | `"message":"type alias cycle through `A`"` |
| Type without finite values | rejected (T0) | `TypeMismatch` | `"message":"type `T` has no finite values"` |
| Monomorphisation deeper than 32 | rejected (T0) | `TypeMismatch` | `"message":"monomorphisation of `Nest` exceeds depth 32"` |
| More than 64 fields / 65,536 variants | rejected (T0) | `TypeMismatch` | `"message":"`Big` has 70 fields; the limit is 64"` |
| Non-capability in `uses` | rejected (T0) | `TypeMismatch` | `"message":"`SessionToken` in uses is not a capability"` |
| Capability op type cannot cross the boundary / duplicate op | rejected (T0) | `TypeMismatch` | `"message":"type `SessionToken` of operation `sign` cannot cross the provider boundary"` |
| Non-deterministic op, spec call or construction in a contract | rejected (T0) | `TypeMismatch` | `"message":"operation `Auth::Clock::now` is not deterministic and cannot appear in a contract"` |
| Contract clause not `Bool` | rejected (T0) | `TypeMismatch` | `"message":"contract clause has type `U64`, expected `Bool`"` |
| Invalid or duplicate waiver | rejected (T0) | `TypeMismatch` | `"message":"waiver `nonce` does not name a part of the result type `SessionToken`"` |
| Struct field or variant field neither mentioned nor waived; no variant test | rejected (T0) | `UnconstrainedOutput` | see Example |
| `match` misses a variant | rejected (T0) | `NonExhaustiveMatch` | `"message":"match on `Result[U64, Bytes]` does not cover Err"` |
| Function name used as a value (SA-12) / entry called | rejected (T0) | `TypeMismatch` | `"message":"entry points cannot be called"` |
| Expression nesting deeper than 256 | rejected (T0) | `TypeMismatch` | `"message":"expression nesting exceeds 256"` |
| `check_spec` called with a DefId that is not a spec or entry | `SpecCheck` with diagnostic | `UnknownName` | `"message":"no spec or entry with this identity"` |
| `par_map` names something other than a spec | rejected (T0) | `InvalidParMap` | ``"message":"`par_map` needs a spec; `Auth::CryptoCap::hmac_sha256` is a capability operation"`` |
| Mapped spec takes other than one parameter | rejected (T0) | `ArityMismatch` | ``"message":"`Auth::check` takes 2 parameter(s); `par_map` passes 1"`` |
| Vector element type differs from the parameter type | rejected (T0) | `TypeMismatch` | ``"message":"`par_map` over `Vec[U64]` needs `Vec[Bytes]` for `Auth::verify_one`"`` |
| Mapped spec has a `requires` clause | rejected (T0) | `InvalidParMap` | ``"message":"`Auth::verify_one` has a requires clause; `par_map` needs a spec without one"`` |

`lower` has no error rows: it is total on any `TImpl` produced by `check_impl`. An IR verifier failure on its output is detected by the caller and reported as a daemon failure (A-17).

#### Acceptance Criteria (Gherkin)

```gherkin
Feature: cprime-types

  Scenario: Happy path — concept example checks and lowers
    Given the auth.cprime and generate_token.opq of the Example section
    When check_spec, check_impl and lower run
    Then no diagnostic is produced
    And the lowered function has exactly one block, one Bin Add, two Intrinsic U64Be, one Perform and one MakeStruct
    And ir_verify::verify_function(.., IrStage::Lowered) returns Ok
    And node_map maps the Bin Add value to node 6, the Perform value to node 16 and the returned value to node 19
    And node_map maps v0, v1, v2 to the parameter nodes 1, 2, 3 and has no entry for the Const v3

  Scenario: Error path — struct field neither mentioned nor waived
    Given generate_token's ensures without the clause mentioning result.sig
    When check_spec runs
    Then exactly one UnconstrainedOutput is returned with clause "result.sig"

  Scenario: Edge case — waiver accepts the gap
    Given spec make_session(user_id U64) Session with "unconstrained nonce" and "ensures result.user_id == user_id"
    And struct Session { user_id U64; nonce Bytes }
    When check_spec runs
    Then no diagnostic is produced

  Scenario: Error path — sum result without a variant test (ADR-026)
    Given spec parse(b Bytes) Result[U64, U8] with "ensures result.Ok.value > 0"
    When check_spec runs
    Then an UnconstrainedOutput with clause "result is Ok" is returned
    And an UnconstrainedOutput with clause "result.Err.error" is returned

  Scenario: Edge case — sum result fully constrained
    Given the same spec with "ensures result is Ok ==> result.Ok.value > 0" and "ensures result is Err ==> result.Err.error == 1"
    When check_spec runs
    Then no diagnostic is produced

  Scenario: Error path — match on Result missing Err (Phase 1 exit criterion)
    Given an implementation whose body is (match x [(Ok v) v]) where x: Result[U64, U8]
    When check_impl runs
    Then one NonExhaustiveMatch is returned at the match node with message "match on `Result[U64, U8]` does not cover Err"

  Scenario: Error path — wrong parameter count
    Given (impl Auth::generate_token [uid now] ...)
    When check_impl runs
    Then exactly one ArityMismatch is returned at node 0 and timpl is None

  Scenario: Error path — integer literal out of range
    Given a body (+ x 256) where x: U8
    When check_impl runs
    Then a TypeMismatch "literal 256 does not fit in `U8`" is returned at the literal's node

  Scenario: Error path — type with no finite values
    Given struct T { t T } used as a spec parameter type
    When check_spec runs
    Then a TypeMismatch "type `T` has no finite values" is returned

  Scenario: Edge case — recursive sum type is accepted
    Given enum List { Nil; Cons { head U64; tail List } }
    When check_spec runs for spec sum(l List) U64
    Then no diagnostic is produced

  Scenario: Error path — non-deterministic operation in a contract
    Given capability Clock { now() U64 } and "ensures result == Clock::now()"
    When check_spec runs
    Then a TypeMismatch naming "is not deterministic" is returned

  Scenario: Edge case — monomorphisation identity is stable
    Given two specs both using Result[U64, Bytes]
    When check_spec runs for both
    Then both result types are Type::Sum of the same DefId
    And that DefId equals blake3("cprime.mono.v1\0" ‖ Result DefId ‖ 0x03 ‖ 0x05)

  Scenario: Edge case — short-circuit lowering
    Given the body (and (> x 0) (< (/ 10 x) 5)) with x: U64
    When lower runs
    Then the Div instruction is in a block reached only from the true edge of the Branch on (> x 0)

  Scenario: Edge case — join parameter of an if gets the if's node
    Given spec clamp(x U64) U64 with no requires and "ensures result <= x"
    And the implementation (impl Demo::clamp [x] (if (> x 0) x 0))
    When check_impl and lower run
    Then the lowered function has a join block with one U64 parameter
    And node_map maps that join parameter to node 2, the if form
    And node_map maps the Bin Gt value to node 3 and the parameter x to node 1
    And node_map has no entry for either Const U64(0)
    And ir_verify::verify_function(.., IrStage::Lowered) returns Ok

  Scenario: Happy path — parallel map types and lowers
    Given Auth::verify_one(t Bytes) Bool with no requires
    And an implementation body (par_map Auth::verify_one tokens) with tokens: Vec[Bytes]
    When check_impl and lower run
    Then the body has type Vec[Bool]
    And the lowered function contains exactly one Inst::ParMap

  Scenario: Error path — par_map names a capability operation
    Given an implementation body (par_map Auth::CryptoCap::hmac_sha256 keys)
    When check_impl runs
    Then an InvalidParMap "`par_map` needs a spec; `Auth::CryptoCap::hmac_sha256` is a capability operation" is returned

  Scenario: Error path — mapped spec takes two parameters
    Given Auth::check(a, b Bytes) Bool
    And an implementation body (par_map Auth::check tokens)
    When check_impl runs
    Then an ArityMismatch "`Auth::check` takes 2 parameter(s); `par_map` passes 1" is returned

  Scenario: Error path — element type differs from the parameter type
    Given Auth::verify_one(t Bytes) Bool and xs: Vec[U64]
    And an implementation body (par_map Auth::verify_one xs)
    When check_impl runs
    Then a TypeMismatch "`par_map` over `Vec[U64]` needs `Vec[Bytes]` for `Auth::verify_one`" is returned

  Scenario: Error path — mapped spec has a requires clause
    Given Auth::verify_one(t Bytes) Bool with "requires len(t) > 0"
    And an implementation body (par_map Auth::verify_one tokens)
    When check_impl runs
    Then an InvalidParMap "`Auth::verify_one` has a requires clause; `par_map` needs a spec without one" is returned
```

#### Performance, Security, Observability

- **Performance targets:** for one implementation of ≤ 200 IR instructions with a warm Salsa database (`check_spec` memoised), `check_impl` ≤ 2 ms p95 and `lower` ≤ 1 ms p95 on the CI reference host, both inside the T0 ≤ 5 ms and T1 ≤ 20 ms budgets (G5). Cold `check_spec` for a spec with ≤ 20 reachable type definitions ≤ 2 ms p95. Memory: ≤ 1 MiB transient per `check_impl` call.
- **Security:** inputs are untrusted agent source. Depth guard (E-4) prevents stack exhaustion; monomorphisation depth (T-5) prevents unbounded instance creation; no input value is executed. No file or network access; the crate reads only through `Db`.
- **Observability:** one `debug` log line per public call (fields `fn`, `spec`, `diagnostics`, `duration_us`, target `cprime_types`). Timing of the T0/T1 stages is measured by `cprime-daemon`, not here. No metrics or spans.

#### Gaps

None.
