# Component spec — cprime-daemon

> Spec set: cprime
> Status: Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> Last updated: 2025-12-20

### SPEC: cprime-daemon
**File:** `crates/cprime-daemon/src/lib.rs` (modules `cli.rs`, `transport.rs`, `rpc.rs`, `mcp.rs`, `db.rs`, `sync.rs`, `pipeline.rs`, `persist.rs`, `log.rs`, `assurance.rs`) | **Package:** `cprime-daemon` | **Phase:** 3 (`get_context`, `check_impl`, MCP, persistence, sync), 4 (`submit_impl`) | **Dependencies:** `cprime-core`, `cprime-syntax`, `cprime-types`, `cprime-effects`, `cprime-memory`, `cprime-codegen`, `cprime-verify`, `cprime-provider`; third-party `tokio`, `serde`, `serde_json`, `libc`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprime-daemon` is the library behind every project's `cprimed` executable. Its entry function `run` is called by the generated `main` of the project-specific runtime with the statically linked provider registry. It reads JSON-RPC 2.0 requests line by line from standard input, serves the three C′ methods (`get_context`, `check_impl`, `submit_impl`) and the MCP tool surface over the same stream, keeps an in-memory Salsa database that mirrors the project files, drives the T0–T3 pipeline across the compiler crates, and persists an accepted implementation to disk by atomic rename before replying. It also exposes `CprimeDatabase` and the T0/T1 pipeline functions to `cprime-cli`, so the release build re-checks implementations with the same code.

#### Approach

A single-threaded request loop: a `tokio` current-thread runtime owns standard input, standard output and the signal streams; each request's compiler work runs on one blocking worker thread (`tokio::task::spawn_blocking`) so that a signal can abandon it (SA-27), and the loop awaits it before reading the next line, giving strict arrival-order execution (SA-05). The database is a Salsa database whose inputs are refreshed by a metadata scan at the start of every request (SA-06). Rejected alternatives: a multi-threaded runtime with per-spec locks (A-3 as originally written) — unnecessary with one stdio client and harder to reason about; a file-watcher thread — adds a dependency and makes "what the request saw" depend on event timing.

#### Shared Context

Types copied from `phase-2-architecture.md` §2C/§2E. Only the members used here are reproduced.

```rust
// cprime_core::ids
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct Symbol(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct DefId(pub [u8; 32]);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct CapId(pub u32);   // CapId(0) = Diverge
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, serde::Serialize)] #[serde(transparent)]
pub struct NodeId(pub u32);

// cprime_core::diag
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)]
pub enum DiagnosticKind {
    ParseError, UnknownName, ArityMismatch, TypeMismatch, UnauthorizedEffect,
    MissingDecreases, NonExhaustiveMatch, UnconstrainedOutput, InvalidParMap,
    ContractViolation, ReachableTrap, NonDecreasing, VerificationInconclusive,
    NonDeterministicProvider,
    InvalidServer,
    VacuousContract, VacuityUnknown,                    // (D37)
}
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")]
pub enum Tier { T0, T1, T2, T3 }
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")]
pub enum Severity { Error, Note }                       // (D37) Note only on VacuityUnknown
#[derive(Clone, PartialEq, Eq, serde::Serialize)]
pub struct Span { pub file: String, pub line: u32, pub column: u32 }   // "<submission>" for the submitted source
#[derive(Clone, serde::Serialize)]
pub struct Diagnostic {
    pub kind: DiagnosticKind, pub tier: Tier, pub severity: Severity, pub node: NodeId,
    #[serde(skip_serializing_if = "Option::is_none")] pub span: Option<Span>,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")] pub clause: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")] pub counterexample: Option<Counterexample>,
    #[serde(skip_serializing_if = "Option::is_none")] pub reason: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")] pub witness: Option<WitnessTrace>,   // (D22)
    #[serde(skip_serializing_if = "Option::is_none")] pub concrete: Option<ConcreteCheck>, // (D43) passed through unchanged
}
#[derive(Clone, serde::Serialize)]
pub struct ConcreteCheck { pub status: ConcreteStatus, #[serde(skip_serializing_if = "Option::is_none")] pub reason: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")] pub witness: Option<WitnessTrace> }   // (D43) this crate fills witness step spans and names as for witness (D26)
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")]
pub enum ConcreteStatus { Concrete, Abstract }
#[derive(Clone)]
pub struct Counterexample { pub inputs: Vec<(String, Value)>, pub expected: Vec<(String, Value)>, pub actual: Vec<(String, Value)> }
// Serialised as {"inputs": {...}, "expected": {...}, "actual": {...}} using the Value JSON encoding, vector order kept.
// (D22) Witness trace; enums serialised lower-case snake case; values in the Value JSON encoding.
#[derive(Clone, serde::Serialize)]
pub struct WitnessTrace { pub status: WitnessStatus, pub steps: Vec<TraceStep>, pub omitted: u32 }
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "snake_case")]
pub enum WitnessStatus { Reproduced, TrapFirst, NotReproduced }
#[derive(Clone, serde::Serialize)]
pub struct TraceStep {
    pub node: NodeId,
    #[serde(skip_serializing_if = "Option::is_none")] pub span: Option<Span>,    // filled here (D26)
    #[serde(skip_serializing_if = "Option::is_none")] pub name: Option<String>,  // filled here (D26)
    pub kind: StepKind, pub value: Value,
    #[serde(skip_serializing_if = "Option::is_none")] pub operands: Option<Vec<Value>>,
    pub source: StepSource,
}
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "snake_case")]
pub enum StepKind { Param, Compute, Construct, Call, Perform, ParMap, Branch, Match, Return, Trap }
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "snake_case")]
pub enum StepSource { Input, Computed, Solver, Generated, Provider }

// cprime_core::ast (D26) — called once per rejected submission that carries a witness
pub struct NodeIndex { pub spans: Vec<Span>, pub names: BTreeMap<NodeId, String> }
// spans[n] = span of the AST node numbered n (len == node_count); names maps each parameter node and
// each `let` binder's bound-expression node to its identifier.
pub fn node_index(imp: &ast::Impl) -> NodeIndex;

// cprime_core::evidence
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")]
pub enum Evidence { Proved, Tested, None }
#[derive(Clone, Copy, PartialEq, Eq)] pub enum Assurance { Proved, Tested }
pub struct VerificationResult {
    pub accepted: bool, pub evidence: Evidence, pub assumed: Vec<String>,
    pub diagnostics: Vec<Diagnostic>, pub t2_us: u64, pub t3_us: Option<u64>,
    pub witness_us: Option<u64>,                              // (D25, D43) Some iff at least one witness or concrete replay ran
    pub vacuity: Option<VacuityOutcome>,                      // (D37) Some iff the vacuity check ran in this call
}
#[derive(Clone)]
pub enum VacuityOutcome { Satisfiable, Vacuous { diagnostic: Diagnostic }, Unknown { reason: &'static str } }   // (D37, D38)
#[derive(Clone, PartialEq, Eq, serde::Serialize)]
pub struct EffectiveEvidence { pub own: Evidence, pub effective: Evidence, pub tested: Vec<String>,
    pub unaccepted: Vec<UnacceptedCallee>, pub assumed: Vec<String> }                       // (D39)
#[derive(Clone, PartialEq, Eq, serde::Serialize)]
pub struct UnacceptedCallee { pub spec: String, pub status: String }                        // "absent" | "stale" | "invalid"

// cprime_core::accept
pub struct AcceptanceHeader { pub context: [u8; 32], pub evidence: Evidence, pub assumed: Vec<String> }
// ";; cprime-accepted v1 context=<64 lower-case hex> evidence=<proved|tested> assumed=<a,b,c | ->"
pub fn format_header(h: &AcceptanceHeader) -> String;          // without trailing "\n"
pub fn parse_header(first_line: &str) -> Result<AcceptanceHeader, HeaderError>;
pub enum HeaderError { Missing, Malformed { reason: &'static str } }
#[derive(Clone, PartialEq, Eq, serde::Serialize)] #[serde(tag = "status", rename_all = "lowercase")]
pub enum ImplStatus { Absent, Current { evidence: Evidence, assumed: Vec<String> }, Stale, Invalid { reason: String } }

// cprime_core::config (fields used here)
pub struct ProjectConfig { pub project: ProjectSection, pub providers: Vec<ProviderEntry>, pub limits: Limits,
    pub verify: VerifyConfig, pub fuzz: FuzzConfig, pub log: LogConfig, pub runtime: RuntimeConfig,
    pub release: ReleaseConfig }                          // (D29) [release]; read by cprime-cli, not by cprimed
pub struct ProjectSection { pub name: String }
pub struct ProviderEntry { pub crate_name: String, pub version: Option<String>, pub path: Option<PathBuf>, pub provider: String }
pub struct VerifyConfig { pub obligation_timeout_ms: u64, pub submission_budget_ms: u64, pub solver_threads: Option<u32>,
    pub witness_max_steps: u32,                           // (D29) 0 … 1024, default 64; passed through to cprime-verify
    pub provider_wait_ms: u64 }                           // (D41) 10 … 600,000, default 2000; passed through to cprime-verify
pub struct ReleaseConfig { pub call_chain: bool }         // (D29) default false
pub struct FuzzConfig { pub iterations: u64, pub time_budget_ms: u64, pub call_fuel: u64, pub call_memory_bytes: u64, pub seed: Option<u64> }
pub struct LogConfig { pub level: LogLevel }
pub enum LogLevel { Error, Warn, Info, Debug, Trace }
pub fn load_config(root: &Path, overrides: &[(String, String)]) -> Result<ProjectConfig, ConfigError>;
pub enum ConfigError { NotFound { root: PathBuf }, Io { path: PathBuf, message: String },
    Toml { line: u32, column: u32, message: String }, Invalid { key: String, message: String }, UnknownOverride { key: String } }

// cprime_core::db
#[salsa::input] pub struct SourceFile { pub path: String, pub text: String }
#[salsa::input] pub struct ProjectInputs { pub files: Vec<SourceFile>, pub config: ProjectConfig, pub providers: Vec<ProviderInfo> }
pub enum BoundaryTy { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<BoundaryTy>), Result(Box<BoundaryTy>, Box<BoundaryTy>) }
pub struct ProviderOpInfo { pub name: String, pub params: Vec<BoundaryTy>, pub ret: BoundaryTy, pub deterministic: bool, pub flat_index: u32,
    pub vectors: u32 }                                    // (D40) known-answer vector count
pub struct ProviderInfo { pub path: String, pub crate_name: String, pub crate_version: String, pub ops: Vec<ProviderOpInfo>, pub thread_safe: bool, pub transport: Option<TransportInfo> }
#[derive(Clone, PartialEq, Eq, Hash)]
pub struct TransportInfo { pub version: u32, pub request: Vec<(String, BoundaryTy)>, pub response: Vec<(String, BoundaryTy)> }   // ADR-051, SA-66
#[salsa::db]
pub trait Db: salsa::Database {
    fn inputs(&self) -> ProjectInputs;
    fn interner(&self) -> &Interner;
    fn parse_module(&self, file: SourceFile) -> Arc<ParsedModule>;
    fn accepted_impl(&self, spec: DefId) -> Option<Arc<AcceptedImpl>>;
    fn impl_status(&self, spec: DefId) -> ImplStatus;
}
pub struct ParsedModule { pub module: Option<ast::Module>, pub diagnostics: Vec<Diagnostic> }
pub struct AcceptedImpl { pub file: String, pub header: AcceptanceHeader, pub timpl: TImpl, pub function: ir::Function }

// cprime_core::program
pub struct Program {
    pub functions: BTreeMap<DefId, ir::Function>, pub layouts: Layouts, pub schema: TypeSchema,
    pub borrows: BTreeMap<DefId, BorrowSig>, pub op_targets: BTreeMap<(CapId, Symbol), OpTarget>,
    pub unimplemented: BTreeSet<DefId>,
}
pub struct BorrowSig { pub borrowed: Vec<bool> }
pub enum OpTarget { Provider { flat_index: u32 }, Stub }

// cprime_core::tast (members used)
pub struct TSpec { pub def: DefId, pub name: String, pub params: Vec<(Symbol, Type)>, pub ret: Type,
    pub uses: BTreeSet<CapId>, pub assurance: Assurance, pub waivers: Vec<Waiver>,
    pub requires: Vec<TClause>, pub ensures: Vec<TClause>, pub is_entry: bool }
pub struct TImpl { pub spec: DefId, pub params: Vec<(Symbol, NodeId)>, pub decreases: Option<TDecreases>, pub body: TExpr, pub node_count: u32 }
pub struct TerminationInfo { pub component: BTreeSet<DefId>, pub diverges: bool, pub edges: Vec<CallEdge>, pub diagnostics: Vec<Diagnostic> }
pub struct CallEdge { pub caller: DefId, pub callee: DefId, pub node: NodeId, pub caller_is_candidate: bool }

// cprime_core::namespace / resolve / context
pub enum DefKind { Type, Capability, Spec, Entry, Prelude }
pub struct NamespaceTable { pub entries: BTreeMap<String, NsEntry> }
pub struct NsEntry { pub def: DefId, pub kind: NsKind, pub file: String, pub span: Span }
pub enum NsKind { Type, Capability, Spec, Entry, Server, Variant { sum: DefId, tag: u32 } }
#[derive(serde::Serialize)]
pub struct ContextResult { pub spec: String, pub impl_file: String, pub implementation: ImplStatus,
    pub signature: String, pub definitions: Vec<ContextDef> }
#[derive(serde::Serialize)] pub struct ContextDef { pub name: String, pub kind: DefKind, pub source: String }
pub enum ContextError { UnknownSpec { name: String }, ModuleHasErrors { file: String, diagnostics: Vec<Diagnostic> } }

// cprime_provider (members used)
pub enum BoundaryType { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<BoundaryType>), Result(Box<BoundaryType>, Box<BoundaryType>) }
pub type OpFn = unsafe extern "C" fn(args: *const u64, rt: *mut RtContext) -> u64;
pub struct OpDecl { pub name: &'static str, pub params: Vec<BoundaryType>, pub ret: BoundaryType, pub deterministic: bool, pub func: OpFn,
    pub vectors: &'static [KnownAnswer] }                 // (D40)
// (D40) sdk::check_vectors(registry: &ProviderRegistry) -> Vec<VectorFailure>;
//       VectorFailure { pub provider: &'static str, pub op: &'static str, pub index: u32, pub detail: String }
pub struct ProviderDecl { pub path: &'static str, pub crate_name: &'static str, pub crate_version: &'static str, pub ops: Vec<OpDecl>, pub thread_safe: bool, pub transport: Option<TransportDecl> }
pub struct ProviderRegistry { pub providers: Vec<ProviderDecl> }   // built-ins (Crypto, Clock, Random, Log), then cprime.toml order
impl ProviderRegistry { pub fn flat_ops(&self) -> Vec<(&ProviderDecl, &OpDecl)>; pub fn describe_json(&self) -> serde_json::Value; }
```

Cross-crate functions called (§2E, verbatim):

| Crate | Function |
|---|---|
| `cprime-syntax` | `parse_cprime(file: &str, text: &str) -> ParsedModule`; `parse_opq(file: &str, text: &str) -> (Option<ast::Impl>, Vec<Diagnostic>)` |
| `cprime-core` | `namespace::namespace_table(db: &dyn Db) -> Arc<NamespaceTable>`; `context::get_context(db: &dyn Db, spec_name: &str) -> Result<ContextResult, ContextError>`; `accept::context_hash(db: &dyn Db, spec: DefId, deps: &BTreeSet<DefId>) -> [u8; 32]`; `accept::format_header`; `accept::parse_header`; `ir_verify::verify_function(f: &ir::Function, layouts: &Layouts, stage: IrStage) -> Result<(), IrVerifyError>`; `paths::impl_file(spec_qualified: &str) -> String`; `config::load_config` |
| `cprime-types` | `check_spec(db, def) -> Arc<SpecCheck>`; `check_impl(db, spec, imp) -> ImplCheck`; `capability_ops(db, cap) -> Arc<Vec<CapOpSig>>`; `lower(db, spec, imp) -> (ir::Function, Layouts)`; `verification_deps(db, spec, imp) -> BTreeSet<DefId>` |
| `cprime-effects` | `check_effects(db, spec, imp) -> Vec<Diagnostic>`; `termination(db, spec, imp) -> TerminationInfo`; `check_bindings(db) -> BindingCheck` |
| `cprime-memory` | `infer_borrows(functions, fixed) -> BTreeMap<DefId, BorrowSig>`; `insert_rc(f, layouts, borrows) -> Result<ir::Function, IrVerifyError>` |
| `cprime-codegen` | `jit::JitModule::compile(program: &Program, registry: &ProviderRegistry) -> Result<JitModule, CodegenError>` |
| `cprime-verify` | `try_verify(input: VerifyInput) -> Result<VerificationResult, VerifyError>` (struct `VerifyInput` as in §2E, including `interner`; D5; taken by value, owning the `JitModule`, with `program` and `registry` as `Arc` and `vacuity` from the cache, D38, D41) |
| `cprime-provider` | `sdk::check_vectors(registry: &ProviderRegistry) -> Vec<VectorFailure>` (D40) |
| `cprime-core` | `hash::impl_def_id(db: &dyn Db, imp: &ast::Impl) -> Result<DefId, Diagnostic>` (D10) |
| `cprime-core` | `ast::node_index(imp: &ast::Impl) -> NodeIndex` (D26) |

Configuration keys read (§2D): `project.name`, `providers[]`, `verify.*` (including `verify.witness_max_steps`, D29), `fuzz.*` (passed through to `cprime-verify`), `log.level`. `release.call_chain` is parsed with the rest of `cprime.toml` but not used by `cprimed`.

**Witness traces (RFC 03, ADR-047, D22, D25, D26).** `cprime-verify` attaches a `witness` to each `ContractViolation` and `ReachableTrap` diagnostic that carries a counterexample, with every step's `span` and `name` left `None`, and reports the replay time as `VerificationResult.witness_us`. This crate fills `span` and `name` from the parsed submission, which only it holds (`FrontEnd.imp`; the typed AST has no spans and no binder names), and reports `witness_us` as `timings_us.witness`. It adds no step, removes no step and changes no value.

#### Public Interface

```rust
// crates/cprime-daemon/src/lib.rs
pub fn run(registry: ProviderRegistry) -> std::process::ExitCode;

pub mod db {
    pub struct CprimeDatabase { /* see Data Model */ }
    impl CprimeDatabase {
        /// (D9) `cprimed` passes `providers` derived from its linked registry (one `ProviderInfo` per
        /// `ProviderDecl`, `flat_index` from `flat_ops()`) and `Some(registry)`; `cprimec` passes the list read
        /// from `.cprime/providers.json` and `None`. With `None`, `back_end` (JIT) is unavailable.
        pub fn open(root: &Path, config: ProjectConfig, providers: Vec<ProviderInfo>, registry: Option<ProviderRegistry>) -> Result<CprimeDatabase, OpenError>;
        pub fn sync(&mut self) -> SyncReport;
        pub fn root(&self) -> &Path;
        pub fn config(&self) -> &ProjectConfig;
        pub fn manifest_changed(&self) -> bool;      // cprime.toml metadata differs from start-up
    }
    impl cprime_core::db::Db for CprimeDatabase { /* hooks: Internal Logic §D */ }
    pub struct SyncReport { pub added: u32, pub changed: u32, pub removed: u32, pub duration_us: u64 }
    pub enum OpenError { CreateStateDir { path: PathBuf, message: String }, Io { path: PathBuf, message: String } }
}

pub mod pipeline {
    pub fn front_end(db: &CprimeDatabase, spec_name: &str, source: &str) -> FrontEnd;
    pub fn back_end(db: &CprimeDatabase, fe: &FrontEnd) -> BackEnd;

    pub struct FrontEnd {
        pub spec_name: String,
        pub spec: Option<Arc<TSpec>>,              // Some once the spec resolves and its declaration type-checks
        pub schema: Option<TypeSchema>,
        pub imp: Option<ast::Impl>,
        pub timpl: Option<TImpl>,                   // Some iff no T0 diagnostic
        pub termination: Option<TerminationInfo>,
        pub diagnostics: Vec<Diagnostic>,          // T0, ordered (Internal Logic §E.9)
        pub t0_us: u64,
    }
    pub struct BackEnd {
        pub function: Option<ir::Function>,        // candidate IR before RC insertion
        pub program: Option<Arc<Program>>,         // candidate + linked accepted callees, RC inserted (Arc, D41)
        pub jit: Option<JitModule>,                // moved into VerifyInput by submit_impl (D41)
        pub failure: Option<DaemonFailure>,         // compiler bug or JIT failure (A-17)
        pub t1_us: u64,
    }
    #[derive(Clone)]
    pub struct DaemonFailure { pub reason: FailureReason, pub detail: String }
    #[derive(Clone, Copy, serde::Serialize)] #[serde(rename_all = "snake_case")]
    pub enum FailureReason { IrVerifier, Codegen, SolverInit, VerifierFailure, UnsupportedConstruct, WriteFailed, Io, Internal }   // serialised snake_case; UnsupportedConstruct (D44)
}

pub mod assurance {
    /// (D39, ADR-053) Own and effective evidence of `spec`'s accepted implementation; None unless impl_status(spec) is Current.
    pub fn effective(db: &dyn cprime_core::db::Db, spec: DefId) -> Option<EffectiveEvidence>;
}
```

**Command line of `cprimed`** (parsed by `run` without `clap`; the argument set is small):

| Flag | Value | Default | Meaning |
|---|---|---|---|
| `--project <dir>` | path | current directory | Project root; must contain `cprime.toml` (no upward search) |
| `--config <key>=<value>` | repeatable | none | Configuration override (§2D) |
| `--log-level <level>` | `error`\|`warn`\|`info`\|`debug`\|`trace` | `log.level` | Overrides `log.level` |
| `--describe-providers` | — | — | Print `registry.describe_json()` as one line to stdout, exit 0; reads no project |
| `--check-providers` | — | — | (D40) Run `sdk::check_vectors(&registry)`; print one stderr line per failure, `cprimed: provider <path> operation <op> fails known-answer vector <index>: <detail>`; exit 1 if any failed, else 0; reads no project |
| `--version` | — | — | Print `cprimed <crate version>` and exit 0 |

**Exit codes of `cprimed`:** 0 — normal end (end of input, `SIGINT`, `SIGTERM`, `--describe-providers`, `--version`, `--check-providers` with no failure); 1 — `--check-providers` found a failing vector (D40); 2 — usage error or `ConfigError`; 3 — `.cprime/` cannot be created; 4 — start-up failure: invalid registry, runtime does not match `cprime.toml`, initial sync I/O failure, Z3 context creation failure.

**JSON-RPC methods.** Transport: one UTF-8 JSON object per line on stdin; one reply object per line on stdout, written with a single `write_all` of the serialised object plus `\n`, then flushed. Nothing else is ever written to stdout.

| Method | `params` (exact; unknown members rejected) | `result` |
|---|---|---|
| `get_context` | `{"spec": string}` | `ContextResult` |
| `check_impl` | `{"spec": string, "source": string}` | `{"status": "ok"\|"rejected", "timings_us": {...}, "diagnostics": [...]}` |
| `submit_impl` | `{"spec": string, "source": string}` | `{"status": "accepted"\|"rejected", "evidence": "proved"\|"tested"\|"none", "assumed": [string] (present iff accepted), "effective": EffectiveEvidence (present iff accepted; D39), "impl_file": string (present iff accepted), "timings_us": {...}, "diagnostics": [...]}`. An accepted reply's `diagnostics` holds only `note` diagnostics (D37) and is usually empty. |
| `initialize` (MCP) | MCP `InitializeRequest` params | `{"protocolVersion", "capabilities": {"tools": {"listChanged": false}}, "serverInfo": {"name": "cprimed", "version": <crate version>}}` |
| `notifications/initialized` (MCP) | any | no reply (notification) |
| `ping` (MCP) | absent or `{}` | `{}` |
| `tools/list` (MCP) | absent or `{"cursor"?: string}` | `{"tools": [3 tool objects]}` (no `nextCursor`) |
| `tools/call` (MCP) | `{"name": string, "arguments": object}` | `{"content": [{"type": "text", "text": <result serialised as JSON string>}], "structuredContent": <result object>, "isError": false}` |

`spec` must match `^[A-Za-z_][A-Za-z0-9_]*(::[A-Za-z_][A-Za-z0-9_]*)+$`; otherwise `-32602`. `timings_us` contains `sync` and `t0` always, `t1` when T1 ran, `t2` when T2 ran, `t3` when T3 ran, `witness` when any witness trace was built (`VerificationResult.witness_us` is `Some`); values are integer microseconds.

MCP protocol versions supported: `"2025-06-18"`, `"2025-03-26"`, `"2024-11-05"`. `initialize` replies with the client's `protocolVersion` when it is in this list, otherwise with `"2025-06-18"`. `tools/list` and `tools/call` are served whether or not `initialize` has been received (a `warn` log line records the omission once).

MCP tools (names per ADR-018; `inputSchema` is JSON Schema 2020-12 with `"additionalProperties": false`):

| Tool | Description (exact) | `inputSchema.properties` / `required` |
|---|---|---|
| `cprime_get_context` | `Return the contract and every definition reachable from a C′ spec, plus the status of its accepted implementation.` | `spec: {type: string}` / `["spec"]` |
| `cprime_check_impl` | `Run the fast checks (parse, types, effects, termination, completeness, IR, JIT) on an .opq implementation without writing it.` | `spec: {type: string}, source: {type: string}` / `["spec","source"]` |
| `cprime_submit_impl` | `Verify an .opq implementation against its contract and, if accepted, write it to the project.` | `spec: {type: string}, source: {type: string}` / `["spec","source"]` |

**JSON-RPC error codes** (`error.data` always an object):

| Code | `message` | `data` | When |
|---|---|---|---|
| -32700 | `parse error` | `{"detail": <serde_json message>}` | Line is not valid JSON (reply `id: null`) |
| -32600 | `invalid request` | `{"detail": string}` | Not an object; `jsonrpc` ≠ `"2.0"`; `id` is null, boolean, array or object; batch array (`"batch requests are not supported"`); line over 16 MiB (`"line exceeds 16777216 bytes"`) |
| -32601 | `method not found` | `{"method": string}` | Unknown method |
| -32602 | `invalid params` | `{"detail": string}` | Missing, extra or mistyped member; malformed `spec`; unknown tool name |
| -32603 | `internal error` | `{"detail": string}` | Worker thread panicked |
| -32001 | `daemon failure` | `{"reason": FailureReason, "detail": string}` | IR verifier failure (A-17), codegen failure, Z3 context failure, verifier internal failure (`try_verify` `Err`), write failure (A-14), I/O failure reading project files |
| -32002 | `unknown spec` | `{"spec": string}` | `get_context` names no spec or entry |
| -32003 | `module has errors` | `{"file": string, "diagnostics": [Diagnostic]}` | `get_context` target module fails to parse |
| -32004 | `restart required` | `{"detail": "cprime.toml changed since start-up; restart with cprimec daemon"}` | `cprime.toml` metadata changed after start-up (A-10) |
| -32005 | `implementation changed on disk` | `{"impl_file": string}` (project-relative path that changed) | `submit_impl` accepted, but the conflict check of §H.5 found that another `cprimed` changed the target `.opq` or a `.opq` this verification read (SA-05) |

Unknown spec in `check_impl`/`submit_impl` is not an error: it is a rejection with an `UnknownName` diagnostic (ADR-010).

##### Example

Request (one line on stdin):
```json
{"jsonrpc":"2.0","id":7,"method":"submit_impl","params":{"spec":"Auth::generate_token","source":"(impl Auth::generate_token [uid now secret]\n  (let [exp (+ now 3600)\n        msg (++ (u64_be uid) (u64_be exp))\n        sig (perform CryptoCap::hmac_sha256 secret msg)]\n    (SessionToken uid exp sig)))\n"}}
```
Reply (one line on stdout; pretty-printed here):
```json
{"jsonrpc":"2.0","id":7,"result":{
  "status":"accepted","evidence":"proved",
  "assumed":["Auth::CryptoCap::hmac_sha256"],
  "effective":{"own":"proved","effective":"proved","tested":[],"unaccepted":[],"assumed":["Auth::CryptoCap::hmac_sha256"]},
  "impl_file":"auth/generate_token.opq",
  "timings_us":{"sync":310,"t0":1820,"t1":7950,"t2":96400},
  "diagnostics":[]}}
```
File written, `auth/generate_token.opq`:
```lisp
;; cprime-accepted v1 context=3b7e0c1f9a2d4e5f60718293a4b5c6d7e8f90112233445566778899aabbccddee evidence=proved assumed=Auth::CryptoCap::hmac_sha256
(impl Auth::generate_token [uid now secret]
  (let [exp (+ now 3600)
        msg (++ (u64_be uid) (u64_be exp))
        sig (perform CryptoCap::hmac_sha256 secret msg)]
    (SessionToken uid exp sig)))
```
Rejected `check_impl` (unknown name), reply:
```json
{"jsonrpc":"2.0","id":8,"result":{"status":"rejected","timings_us":{"sync":290,"t0":610},
 "diagnostics":[{"kind":"UnknownName","tier":"t0","severity":"error","node":9,
   "span":{"file":"<submission>","line":3,"column":13},"message":"name `u64_le` is not defined"}]}}
```
MCP call:
```json
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"cprime_get_context","arguments":{"spec":"Auth::generate_token"}}}
```
Reply: `{"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"{\"spec\":\"Auth::generate_token\",...}"}],"structuredContent":{"spec":"Auth::generate_token","impl_file":"auth/generate_token.opq","implementation":{"status":"absent"},"signature":"spec generate_token(...) ...","definitions":[...]},"isError":false}}`

#### Internal Logic

**A. Start-up (`run`)**

1. Parse `std::env::args_os()` against the flag table. Unknown flag, missing value, non-UTF-8 argument, or `--config` value without `=` → write `cprimed: usage: <detail>` to stderr, return exit 2. Logging is not yet configured; this is the only unstructured stderr line besides the start-up messages in steps 3 and 5.
2. If `--version`: print and return 0. If `--describe-providers`: validate the registry (step 4), print `registry.describe_json()` serialised on one line plus `\n`, return 0 (exit 4 if invalid). If `--check-providers` (D40): validate the registry (step 4; exit 4 if invalid), call `cprime_provider::sdk::check_vectors(&registry)`, write one stderr line per `VectorFailure` (`cprimed: provider <provider> operation <op> fails known-answer vector <index>: <detail>`), return 1 if any, else 0. Neither flag reads a project.
3. Resolve `--project` to an absolute path (`std::fs::canonicalize`). Call `cprime_core::config::load_config(root, overrides)`. `Err(e)` → stderr `cprimed: config: <e>`, exit 2. Apply `--log-level`. Configure the logger (Logging cross-cutting spec) with the level.
4. Validate the registry: provider paths unique; operation names unique within a provider; every `OpDecl.params`/`ret` contains no `FixedBytes(0)`. Then compare the registry's non-built-in providers (index ≥ 4) against `config.providers`, in order, by (`crate_name`, `path`/`provider`): lengths and every pair must match. Mismatch → log `error` `msg="runtime does not match cprime.toml; run cprimec runtime build"`, exit 4.
5. Create `<root>/.cprime/` (`create_dir_all`). Failure → stderr `cprimed: cannot create <root>/.cprime: <OS message>`, exit 3. No lock is held at start-up: other `cprimed` instances may serve the same project (SA-05).
6. Delete leftover temporary files while holding the project write lock (acquired as in §H.4; lock busy for 5,000 ms → skip the cleanup and log `warn` `msg="cleanup skipped; write lock busy"`): walk the project as in §C and remove every regular file whose name matches `^\..+\.opq\.tmp-[0-9]+$`. Holding the lock guarantees no other `cprimed` is mid-write. Each removal logs `warn` (Graceful Shutdown cross-cutting spec). Release the lock.
7. `CprimeDatabase::open(root, config, providers, Some(registry.clone()))`, where `providers` is built from the registry (one `ProviderInfo` per `ProviderDecl`, each `ProviderOpInfo.flat_index` from `flat_ops()` and `vectors` = `OpDecl.vectors.len()` (D40), `BoundaryType` mirrored to `BoundaryTy`), then `sync()`. I/O failure → log `error`, exit 4.
8. No Z3 warm-up runs at start-up; a Z3 context creation failure surfaces on the first `submit_impl` as `-32001` with reason `solver_init`. Log `info` `msg="cprimed ready"` with fields `root`, `files`, `providers`.
9. Build a `tokio` current-thread runtime (`enable_io`, `enable_time`) and run the serve loop (§B). Return its exit code.

**B. Serve loop**

1. Install `tokio::signal::unix` streams for `SIGINT` and `SIGTERM`.
2. Read stdin through a `BufReader` with a 64 KiB buffer, splitting on `\n`. A line longer than 16,777,216 bytes (excluding `\n`) is discarded up to the next `\n` and answered with `-32600`, `id: null`. A trailing `\r` is removed. Empty or whitespace-only lines are skipped silently.
3. Parse the line with `serde_json::from_slice` into `serde_json::Value`. Failure → `-32700`, `id: null`.
4. Validate the envelope: array → `-32600` (`"batch requests are not supported"`, `id: null`); not an object, `jsonrpc != "2.0"`, `method` absent or not a string → `-32600` with the `id` echoed when it is a number or string, else `null`. `id` present with type null, boolean, array or object → `-32600`, `id: null`.
5. Notifications (no `id`): `notifications/initialized` sets `initialized = true`; `notifications/cancelled` is logged at `debug` and ignored (SA-15: no cancellation); any other notification, including C′ methods sent without `id`, is logged at `warn` (`msg="notification ignored"`, `method`) and not executed. No reply is written.
6. Dispatch by method name. MCP `initialize`, `ping`, `tools/list` are answered on the loop thread. `get_context`, `check_impl`, `submit_impl`, and `tools/call` run the method body inside `tokio::task::spawn_blocking`, which owns `&mut CprimeDatabase` for the duration (the database is moved into the closure and returned with the reply).
7. `tokio::select!` between the worker's completion and either signal. On completion: serialise the reply, write it, flush, log the request line (§F), continue. A worker panic (`JoinError::is_panic`) → `-32603` and the database is rebuilt with `CprimeDatabase::open` + `sync` before the next request; log `error`.
8. On `SIGINT` or `SIGTERM` (at any point, including while a worker runs): log `info` `msg="signal received; exiting"` and call `std::process::exit(0)` without writing a reply (SA-27). No temporary file cleanup is attempted; step A.6 removes leftovers on the next start. Target: exit within 200 ms of signal delivery, which holds because nothing is awaited after the signal.
9. On end of stdin (read returns 0 bytes): the in-flight request, if any, has already completed because execution is sequential; log `info` `msg="input closed; exiting"`, return exit 0.
10. Write errors on stdout (`EPIPE`) → log `error`, return exit 0 (the client has gone).

**C. Source sync (`CprimeDatabase::sync`, SA-06)**

1. Record `start = Instant::now()`.
2. Walk `root` depth-first with `std::fs::read_dir`, sorted by file name. Skip directory entries named `.cprime`, `target`, or starting with `.`. Skip symbolic links (files and directories; logged once per path at `debug`). Collect regular files with extension `pact` or `opq`, plus `root/cprime.toml`.
3. For each collected file, read `metadata()` → (`modified()` as nanoseconds since the Unix epoch, `len()`). Compare with `last_seen: BTreeMap<String, (u128, u64)>` keyed by project-relative path with `/` separators.
4. New or changed paths: `std::fs::read`. Bytes that are not valid UTF-8 → record the path in `non_utf8` and set the input text to the empty string; `parse_module` reports it (§D.2). For `.opq` and `.cprime` files, create or update the `SourceFile` input (`set_text`). Count `added`/`changed`.
5. For `cprime.toml`, do not reload configuration (A-10); if its metadata differs from the start-up value set `manifest_changed = true`.
6. Paths in `last_seen` but not found → remove from `last_seen` and from `ProjectInputs.files`. Count `removed`.
7. If any file was added or removed, call `ProjectInputs::set_files` with the new list sorted by path.
8. Read errors on one file (permission denied, vanished between `read_dir` and `read`) → treat the file as removed, log `warn` with `path` and the OS message. Read errors on `root` itself → return the error to the caller, which replies `-32001` reason `io`.
9. Return `SyncReport { added, changed, removed, duration_us }`; log `debug` `msg="sync"` with the counts.

**D. Database hooks (`impl Db for CprimeDatabase`)**

1. `inputs()` returns the `ProjectInputs` value created in `open`. `interner()` returns the interner (§Data Model); `CapId(0)` is pre-interned as `Diverge` in `open`.
2. `parse_module(file)`: if `file.path` is in `non_utf8` → `ParsedModule { module: None, diagnostics: [ParseError, T0, node 0, span {file, 1, 1}, "file is not valid UTF-8"] }`. Otherwise `cprime_syntax::parse_cprime(&path, &text)`. Memoised by a `#[salsa::tracked]` function `parsed_module_q(db, file)` defined in `db.rs`.
3. `impl_status(spec)` via tracked function `impl_status_q(db, spec)`:
   a. Find the qualified name of `spec` in `namespace_table(db)` (reverse map built once per revision). Not found → `Absent`.
   b. Path = `paths::impl_file(name)`. No `SourceFile` input with that path → `Absent`.
   c. First line of the text → `accept::parse_header`. `Missing` → `Invalid { reason: "acceptance header missing" }`; `Malformed { reason }` → `Invalid { reason }`.
   d. `cprime_syntax::parse_opq(path, text)`; any diagnostic or `None` → `Invalid { reason: "file does not parse" }`.
   e. `check_spec(db, spec)` has diagnostics → `Stale` (the spec no longer type-checks, so the recorded context cannot match). Otherwise `check_impl(db, &spec, &ast)`; diagnostics → `Stale`.
   f. `deps = verification_deps(db, &spec, &timpl)`; `h = context_hash(db, spec, &deps)`. `h == header.context` → `Current { evidence: header.evidence, assumed: header.assumed }`; else `Stale`.
4. `accepted_impl(spec)` via tracked `accepted_impl_q(db, spec)`: if `impl_status(spec)` is `Current`, return `AcceptedImpl { file, header, timpl, function }` where `function` is `cprime_types::lower(db, &spec, &timpl).0` verified by `ir_verify::verify_function(.., IrStage::Lowered)`; a verifier failure here returns `None` and logs `error` (it is a compiler bug, A-17; the dependant request will treat the callee as unimplemented). Any other status → `None`.

5. `assurance::effective(db, spec)` (D39, ADR-053), via tracked `effective_q(db, spec)`:
   a. `impl_status(spec)` not `Current { evidence, assumed }` → `None`.
   b. `own = evidence`; `effective = evidence`; `assumed_set` = `assumed`; `visited = {spec}`; `work` = the `Call.func` and `ParMap.func` targets of `accepted_impl(spec).function` (pre-RC IR), each once.
   c. Pop `d` from `work` (smallest `DefId` first); skip if in `visited`; add to `visited`. Name `n` = qualified name from the namespace table. Match `impl_status(d)`: `Current { evidence: e, assumed: a }` → `effective = min(effective, e)` with `proved > tested > none`; `e == Tested` → push `n` to `tested`; extend `assumed_set` with `a`; push the `Call.func`/`ParMap.func` targets of `accepted_impl(d).function` to `work`. `Absent`, `Stale`, `Invalid { .. }` → push `UnacceptedCallee { spec: n, status: "absent" | "stale" | "invalid" }`; `effective = None`; do not walk `d`. A `d` whose `accepted_impl` is `None` while `Current` (IR verifier failure, A-17) is treated as `invalid`.
   d. Return `Some(EffectiveEvidence { own, effective, tested (sorted), unaccepted (sorted by spec), assumed (sorted, deduplicated) })`. Nothing is stored on disk; the Salsa memo is invalidated when any header or source changes, so re-accepting a callee changes the result without resubmitting the caller (ADR-053).

**E. Pipeline — `front_end` (T0)**

1. `t0_start = Instant::now()`.
2. Look up `spec_name` in `namespace_table(db)`. Absent, or kind not `Spec`/`Entry` → diagnostic `UnknownName`, T0, node 0, span `None`, message ``spec `<name>` is not declared``; return.
3. `module_file` of the spec's module: `parse_module` diagnostics → copy them (they carry `.cprime` spans) and return. Then copy every diagnostic of `cprime_core::namespace::namespace_diagnostics(db)` whose `span.file` is that module file (duplicate qualified names, SA-38) and return if any was copied.
4. `check_spec(db, def)`: copy `diagnostics` (this includes `UnconstrainedOutput` and the ADR-026 rule, SA-13). If `spec` is absent in the result, return.
5. Strip a leading acceptance header line from `source` if the first line starts with `;; cprime-accepted` (an agent may resubmit a file read from disk). `cprime_syntax::parse_opq("<submission>", source)`; copy diagnostics; `None` → return.
6. If the parsed `imp.spec` path, resolved in the spec's module (`resolve::lookup` through `check_impl`), names a different definition from `spec_name` → `TypeMismatch`, node 0, message ``implementation names `<impl path>` but params.spec is `<spec_name>` ``; return.
7. `check_impl(db, &spec, &imp)` → copy diagnostics (`UnknownName`, `ArityMismatch`, `TypeMismatch`, `NonExhaustiveMatch`). If `timpl` is `None`, return.
8. `check_effects(db, &spec, &timpl)` (`UnauthorizedEffect`) and `termination(db, &spec, &timpl)` (`MissingDecreases`, `TypeMismatch` on measure length); copy diagnostics; store `termination`.
9. Order diagnostics: by tier, then by `span` (file, line, column; `None` last), then by `node`, then by kind declaration order. Remove exact duplicates (same kind, node, span, message). If any diagnostic remains, set `timpl = None`.
10. `t0_us = elapsed`. Log `debug` `msg="t0 done"` with `spec`, `diagnostics` count.

**F. Pipeline — `back_end` (T1)**

Runs only when `fe.timpl` is `Some`.

1. `t1_start`. `(function, layouts) = cprime_types::lower(db, spec, timpl)`. `verify_function(&function, &layouts, IrStage::Lowered)`; `Err` → `failure = DaemonFailure { IrVerifier, detail }`; return.
2. Linked set: starting from the candidate, collect called spec `DefId`s (`Inst::Call.func`) transitively; for each callee `c` ≠ candidate: `db.accepted_impl(c)` → `Some(a)` adds `a.function` and continues the walk from it; `None` adds `c` to `unimplemented`. A callee in the candidate's recursive component that is the candidate itself is not looked up.
3. Merge layouts of every linked function (`Layouts` maps are unions; equal keys have equal values by construction of `DefId`).
4. `fixed` = for each `u` in `unimplemented`: `BorrowSig { borrowed: vec![false; param count of check_spec(db,u).spec] }`. `borrows = cprime_memory::infer_borrows(&functions_pre_rc, &fixed)`, then insert `fixed` entries.
5. For each linked function: `insert_rc(f, &layouts, &borrows)`; `Err(e)` → `failure { IrVerifier, e }`; return. Then `verify_function(.., IrStage::RcInserted)`; `Err` → same.
6. `op_targets`: `check_bindings(db)`; for each `(cap, op)` appearing in a `Perform` of any linked function: if `bindings` has `cap` → provider index `p` → `ProviderInfo.ops` entry with `name == op` → `OpTarget::Provider { flat_index }`; otherwise `OpTarget::Stub`. Binding diagnostics are not reported here (they matter for release builds; `cprime-cli` reports them).
7. `schema` = `fe.schema` extended by `check_spec(db, callee).schema` for every linked callee.
8. `JitModule::compile(&program, &registry)`; `Err(e)` → `failure { Codegen, e.to_string() }`.
9. `t1_us = elapsed`. Keep the pre-RC candidate `function` in `BackEnd.function` for `cprime-verify`.

**G. Method bodies**

1. Every C′ method first calls `db.sync()` (timing `sync`). If `db.manifest_changed()` → `-32004`. If `sync` fails → `-32001` reason `io`.
2. `get_context`: `context::get_context(db, spec)`. `Ok(r)` → result `r`. `UnknownSpec` → `-32002`. `ModuleHasErrors` → `-32003`.
3. `check_impl`: `fe = front_end`; if diagnostics → `{"status":"rejected"}`. Else `be = back_end`; `failure` → `-32001`; else `{"status":"ok"}`. `timings_us` per §Public Interface.
4. `submit_impl`: steps of `check_impl`; on T0 rejection reply `status: rejected, evidence: none`. After T1:
   a. Build `callees`: `check_spec(db, c).spec` for every `Call` target of the candidate function and for every member of `termination.component` (D5).
   b. `cap_ops`: `capability_ops(db, cap)` for every `cap` in `spec.uses` minus `CapId(0)`, plus every capability referenced by an `OpApp` in `spec.requires`/`spec.ensures` and by the callees' contracts.
   c. `members`: `db.accepted_impl(m)` for every `m` in `termination.component` other than the candidate (absent members are omitted; they cannot be in the component because the call graph is built over accepted implementations).
   d. `jit: be.jit.take()` iff `spec.assurance == Tested`, else `None` (the module moves into `cprime-verify`, D41). `program: Arc::clone` of `be.program`; `registry: Arc::clone` of the database's registry. `bindings` from `check_bindings`.
   e. `vacuity = self.vacuity_cache.get(&spec.def).cloned()` (D38). `VerifyInput { impl_def, spec, timpl, function, callees, cap_ops, termination, members, program, jit, registry, vacuity, interner: db.interner(), bindings, verify_cfg: &config.verify, fuzz_cfg: &config.fuzz }`. `impl_def = cprime_core::hash::impl_def_id(db, &impl_ast)` (D10); it excludes the acceptance header, comments and local names, so it identifies the candidate for the fuzz seed default (§2D) and is stable across resubmissions of the same implementation. `program.op_targets` maps every operation of a bound capability to `Provider` and every other operation to `Stub` (step 6 of §E; D4).
   f. `res = cprime_verify::try_verify(input)` (D5, D41). `Err(VerifyError::SolverInit(_))` → `-32001` reason `solver_init`; `Err(VerifyError::UnsupportedConstruct { .. })` → `-32001` reason `unsupported_construct` with `detail` = the error text (D44); any other `Err` → `-32001` reason `verifier_failure` with `detail` = the error text (A-17); a worker panic → `-32603`. On `Ok`, when `res.vacuity` is `Some(Satisfiable)` or `Some(Vacuous { .. })`, insert it into `vacuity_cache` under `spec.def` (D38); `Unknown` is not cached.
   g. `res.accepted == false` → `{"status":"rejected","evidence":"none","diagnostics":res.diagnostics}` with `t2`, `t3` timings. Before replying, when any diagnostic carries a `witness` (D26): compute `idx = ast::node_index(fe.imp)` once (`fe.imp` is `Some` whenever T2 ran); for every step of every witness, and of every `concrete.witness` (D43), set `span = idx.spans.get(step.node)` (a node at or beyond `idx.spans.len()` leaves `span` `None` and logs `warn` `msg="witness node out of range"` with `spec` and `node`) and `name = idx.names.get(step.node)`; leave every other field unchanged. When `res.witness_us` is `Some(w)`, add `"witness": w` to `timings_us`.
   h. Accepted → persistence (§H), passing the `.opq` paths of the `members` from step c as the read set. On success, after §H.10, compute `eff = assurance::effective(db, spec.def)` (D39); `None` (impossible once §H.10 has run) logs `error` `msg="effective evidence unavailable"` and uses `EffectiveEvidence { own: res.evidence, effective: res.evidence, tested: [], unaccepted: [], assumed: res.assumed }`. Reply `{"status":"accepted","evidence":res.evidence,"assumed":res.assumed,"effective":eff,"impl_file":path,...,"diagnostics":res.diagnostics}`; `res.diagnostics` holds only `note` diagnostics (D37). On conflict → `-32005` with `data.impl_file` = the changed path; on write failure → `-32001` reason `write_failed`. In both cases no acceptance is reported (A-14).
5. `tools/call`: validate `name` ∈ the three tool names (else `-32602` `"unknown tool <name>"`) and `arguments` is an object (else `-32602`); run the corresponding method body with `arguments` as `params`; a method result becomes the MCP result shape with `isError: false`; a method error is returned as the same JSON-RPC error object (not as `isError: true`).

**H. Persistence (`persist::write_accepted`)**

Outcomes: written, conflict (§G.4.h replies `-32005`), or write failure (`-32001` `write_failed`).

1. `deps = verification_deps(db, &spec, &timpl)`; `context = context_hash(db, spec.def, &deps)`.
2. `header = format_header(&AcceptanceHeader { context, evidence: res.evidence, assumed: res.assumed.clone() })`.
3. Body = submitted source with any leading header line removed (§E.5), CRLF converted to LF, and exactly one trailing `\n`. Content = `header + "\n" + body`.
4. Acquire the project write lock (SA-05): open `<root>/.cprime/lock` with `O_RDWR|O_CREAT|O_CLOEXEC`, mode 0600, and call `libc::flock(fd, LOCK_EX | LOCK_NB)`. On `EWOULDBLOCK`, sleep 10 ms and retry, for at most 5,000 ms in total; still held → write failure with detail `project write lock busy for 5000 ms`. Any other error → write failure with the OS message. The descriptor is closed, releasing the lock, on every path out of steps 5–9.
5. Conflict check (SA-05): for the target path `<paths::impl_file(spec.name)>` and for every path in the read set (§G.4.h), read `metadata()` now as (modification time in nanoseconds, length), or *absent* if the file does not exist. Compare each with its `last_seen` entry, or *absent* if there is none; `last_seen` holds the values recorded by this request's `sync` (§G.1). Any difference → release the lock, log `warn` `msg="write conflict"` with `spec` and `path`, return conflict with that path. Nothing is written.
6. Target `root/<paths::impl_file(spec.name)>`; `create_dir_all(parent)` (mode 0755).
7. Temporary path `parent/.<file name>.tmp-<pid>`; open with `O_WRONLY|O_CREAT|O_TRUNC|O_CLOEXEC`, mode 0644; `write_all`; `sync_all()`; close.
8. `std::fs::rename(tmp, target)`; open `parent` read-only and `sync_all()` it (directory fsync).
9. Any error in 6–8 → remove the temporary file if it exists (errors ignored), log `error` with `path`, release the lock, return write failure.
10. Release the lock. Update the in-memory state: set (or create) the `SourceFile` input text to `content` and refresh `last_seen` for the path from a fresh `metadata()`, so the next sync does not re-read it. Log `info` `msg="implementation accepted"` with `spec`, `path`, `evidence`.

**I. Logging (fields; format owned by the Logging cross-cutting spec)**

One line per completed request at `info`: `target="cprimed::rpc"`, `msg="request completed"`, `request_id` (the JSON-RPC id), `method`, `spec` (when present), `status` (`ok`/`rejected`/`accepted`, or `error:<code>`), `timings_us`, `diagnostics` (count). At `trace` only: `source` (submitted text) and the full `diagnostics` array including counterexamples and witness traces (SA-26). Other targets: `cprimed::sync`, `cprimed::pipeline`, `cprimed::persist`, `cprimed::start`.

#### Data Model

No database schema (A-7). In-memory and on-disk structures owned by this component:

```rust
pub struct CprimeDatabase {
    storage: salsa::Storage<Self>,
    root: PathBuf,                                   // canonical absolute path
    config: ProjectConfig,                           // fixed at start-up (A-10)
    registry: Arc<ProviderRegistry>,                 // cloned from the argument of open
    inputs: ProjectInputs,                           // providers: ProviderInfo mirror of the registry
    interner: Interner,                              // Mutex<(BTreeMap<String, u32>, Vec<String>)> for Symbol; same shape for CapId
    last_seen: BTreeMap<String, (u128, u64)>,        // path → (mtime ns, length)
    files_by_path: BTreeMap<String, SourceFile>,
    non_utf8: BTreeSet<String>,
    manifest_meta: (u128, u64),                      // cprime.toml metadata at start-up
    manifest_changed: bool,
    vacuity_cache: HashMap<DefId, VacuityOutcome>,   // (D38) Satisfiable and Vacuous only; never persisted; dropped with the database
}
```

`ProviderInfo` mirror (built in `open`): for each `ProviderDecl` in registry order: `path`, `crate_name`, `crate_version`, `thread_safe`, and per `OpDecl` a `ProviderOpInfo` with `BoundaryType` mapped one-to-one onto `BoundaryTy`, `flat_index` equal to the operation's position in `registry.flat_ops()`, and `vectors` = `OpDecl.vectors.len()` (D40).

On-disk files written: `<root>/.cprime/lock` (empty, mode 0600), accepted `.opq` files (§H), temporary files `.<name>.opq.tmp-<pid>` (transient). No rollback concern: no destructive schema change exists; an unwanted acceptance is reverted in the user's version control (HLD rollback).

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Line is not JSON | JSON-RPC error | -32700 | `{"jsonrpc":"2.0","id":null,"error":{"code":-32700,"message":"parse error","data":{"detail":"expected value at line 1 column 1"}}}` |
| Batch array received | JSON-RPC error | -32600 | `{"code":-32600,"message":"invalid request","data":{"detail":"batch requests are not supported"}}`, `id: null` |
| Line longer than 16 MiB | JSON-RPC error | -32600 | `data.detail = "line exceeds 16777216 bytes"`, `id: null` |
| `jsonrpc` missing or not `"2.0"`; bad `id` type | JSON-RPC error | -32600 | `data.detail = "jsonrpc must be \"2.0\""` |
| Unknown method | JSON-RPC error | -32601 | `{"code":-32601,"message":"method not found","data":{"method":"compile"}}` |
| Missing/extra/mistyped param, malformed `spec`, unknown tool | JSON-RPC error | -32602 | `data.detail = "unknown member `foo` in params"` |
| Worker panic | JSON-RPC error | -32603 | `data.detail = "worker panicked: <payload>"` |
| IR verifier fails after lowering or RC insertion (A-17) | JSON-RPC error | -32001 | `{"code":-32001,"message":"daemon failure","data":{"reason":"ir_verifier","detail":"block 3: argument count 1 does not match target params 2"}}` |
| JIT compilation fails | JSON-RPC error | -32001 | `data.reason = "codegen"` |
| Z3 context cannot be created | JSON-RPC error | -32001 | `data.reason = "solver_init"` |
| `try_verify` returns `UnsupportedConstruct` (D44) | JSON-RPC error | -32001 | `data.reason = "unsupported_construct"`, `data.detail = "cannot encode smt-lib parse in <def>"` |
| `try_verify` returns any other `Err` (A-17) | JSON-RPC error | -32001 | `data.reason = "verifier_failure"`, `data.detail = "cyclic block graph in Auth::generate_token"` |
| Contract is vacuous (D37), fresh or cached (D38) | result `rejected` | `VacuousContract` | `evidence: "none"`, one diagnostic; a cached outcome costs no solver call |
| Vacuity check undecided (D37) | result unchanged (`accepted` or `rejected`) | `VacuityUnknown` (severity `note`) | the note is the last diagnostic; nothing is cached |
| Accepted caller with a callee that is not current (D39) | result `accepted` | — | `"effective":{"own":"proved","effective":"none","unaccepted":[{"spec":"Lib::b","status":"absent"}],…}` |
| `--check-providers` finds a failing vector (D40) | exit 1 | — | stderr `cprimed: provider host::Crypto operation hmac_sha256 fails known-answer vector 2: result differs` |
| Atomic write fails (A-14) | JSON-RPC error | -32001 | `data.reason = "write_failed"`, `data.detail = "rename …: No space left on device"`; nothing reported as accepted |
| Project write lock held by another `cprimed` for 5,000 ms | JSON-RPC error | -32001 | `data.reason = "write_failed"`, `data.detail = "project write lock busy for 5000 ms"`; nothing written |
| Target `.opq`, or a `.opq` the verification read, changed on disk since the request began (SA-05) | JSON-RPC error | -32005 | `{"code":-32005,"message":"implementation changed on disk","data":{"impl_file":"auth/generate_token.opq"}}`; nothing written |
| Project root unreadable during sync | JSON-RPC error | -32001 | `data.reason = "io"` |
| `get_context` for unknown spec | JSON-RPC error | -32002 | `{"code":-32002,"message":"unknown spec","data":{"spec":"Auth::nope"}}` |
| `get_context` target module fails to parse | JSON-RPC error | -32003 | `data = {"file":"auth.cprime","diagnostics":[…]}` |
| `cprime.toml` changed after start-up | JSON-RPC error | -32004 | `data.detail = "cprime.toml changed since start-up; restart with cprimec daemon"` |
| Spec not declared (`check_impl`/`submit_impl`) | result `rejected` | `UnknownName` | `{"kind":"UnknownName","tier":"t0","severity":"error","node":0,"message":"spec `Auth::nope` is not declared"}` |
| Source names a different spec than `params.spec` | result `rejected` | `TypeMismatch` | `"message":"implementation names `Auth::a` but params.spec is `Auth::b`"` |
| Any T0 diagnostic | result `rejected` | kind from the producing crate | diagnostic array; no T1/T2 run |
| T2/T3 rejection | result `rejected` | `ContractViolation` / `ReachableTrap` / `NonDecreasing` / `VerificationInconclusive` / `NonDeterministicProvider` | `evidence: "none"`, diagnostics from `cprime-verify` |
| T2/T3 rejection whose diagnostics carry witness traces (D25, D26) | result `rejected` | `ContractViolation` / `ReachableTrap` | each `witness` step gains `span` (file `"<submission>"`) and, for parameter and `let`-bound nodes, `name`; `timings_us.witness` present |
| A witness step's `node` is not below the submission's node count (defect in lowering or in `cprime-verify`) | result `rejected` (unchanged) | `ContractViolation` / `ReachableTrap` | that step has no `span` and no `name`; log `warn` `msg="witness node out of range"` with `spec`, `node` |
| Usage error at start | exit code | 2 | stderr `cprimed: usage: unknown flag --porject` |
| `ConfigError` at start | exit code | 2 | stderr `cprimed: config: verify.submission_budget_ms: must be >= obligation_timeout_ms` |
| `.cprime/` cannot be created | exit code | 3 | stderr `cprimed: cannot create /home/u/proj/.cprime: Permission denied (os error 13)` |
| Registry invalid or does not match `cprime.toml` | exit code | 4 | stderr log line `{"level":"error","msg":"runtime does not match cprime.toml; run cprimec runtime build",…}` |
| Initial sync I/O failure | exit code | 4 | log line `msg="initial sync failed"` with `path` and OS message |

#### Acceptance Criteria (Gherkin)

```gherkin
Feature: cprime-daemon

  Background:
    Given a project at "/tmp/p" whose cprime.toml has [project] name = "demo"
    And "auth.cprime" declares module Auth with capability CryptoCap, type SessionToken and spec generate_token as in concept §4.1
    And cprimed is started with "--project /tmp/p"

  Scenario: Happy path — submit an implementation that is proved
    When the client sends submit_impl for "Auth::generate_token" with the concept §4.2 source
    Then the reply result has status "accepted" and evidence "proved"
    And assumed equals ["Auth::CryptoCap::hmac_sha256"]
    And "/tmp/p/auth/generate_token.opq" exists and its first line starts with ";; cprime-accepted v1 context="
    And timings_us has keys sync, t0, t1, t2 and not t3

  Scenario: check_impl never writes
    When the client sends check_impl for "Auth::generate_token" with the concept §4.2 source
    Then the reply result has status "ok"
    And "/tmp/p/auth/generate_token.opq" does not exist

  Scenario: Rejection with counterexample writes nothing
    When the client sends submit_impl whose body uses "(+ now 1000)" instead of "(+ now 3600)"
    Then the reply result has status "rejected" and evidence "none"
    And the first diagnostic has kind "ContractViolation" and clause "ensures: result.expires_at == now + 3600"
    And no file exists under "/tmp/p/auth/"

  Scenario: Unknown spec in submit_impl is a rejection, not an error
    When the client sends submit_impl for "Auth::nope"
    Then the reply has a result, not an error
    And the result diagnostics contain one "UnknownName" with node 0

  Scenario: Unknown spec in get_context is an error
    When the client sends get_context for "Auth::nope"
    Then the reply has error code -32002 and data.spec "Auth::nope"

  Scenario: Human edit is visible to the next request
    Given generate_token has been accepted
    When the human changes the ensures clause "now + 3600" to "now + 7200" in auth.cprime
    And the client sends get_context for "Auth::generate_token"
    Then implementation.status is "stale"

  Scenario: Resubmitting a file that already carries a header
    Given generate_token has been accepted
    When the client sends check_impl with the exact text of auth/generate_token.opq
    Then the reply result has status "ok"

  Scenario: Edge case — batch request rejected
    When the client sends the line "[{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"ping\"}]"
    Then the reply has error code -32600 and id null

  Scenario: Edge case — notification for a C′ method is not executed
    When the client sends submit_impl without an id
    Then no reply is written
    And no file is written

  Scenario: Error path — malformed JSON
    When the client sends the line "{not json"
    Then the reply has error code -32700 and id null

  Scenario: Error path — unknown params member
    When the client sends check_impl with params {"spec":"Auth::generate_token","source":"(impl …)","x":1}
    Then the reply has error code -32602

  Scenario: Error path — write failure reports a daemon failure
    Given the directory "/tmp/p/auth" exists and is not writable
    When the client sends submit_impl for a provable implementation
    Then the reply has error code -32001 and data.reason "write_failed"
    And the reply contains no "accepted" status

  Scenario: Error path — IR verifier failure is a daemon failure (A-17)
    Given cprime-types lowering is replaced in a test build by one that emits a Jump with too few arguments
    When the client sends check_impl for a type-correct implementation
    Then the reply has error code -32001 and data.reason "ir_verifier"

  Scenario: Second daemon on the same project serves its own client
    Given a first cprimed is serving "/tmp/p"
    When a second cprimed is started with "--project /tmp/p"
    And its client sends check_impl for the §4.2 implementation
    Then the second cprimed replies with status "ok"
    And the first cprimed still answers ping

  Scenario: Error path — another session accepted the same spec first
    Given two cprimed instances A and B serve "/tmp/p"
    And B has started submit_impl for "Auth::generate_token"
    When A accepts an implementation of "Auth::generate_token" before B reaches persistence
    Then B replies with error code -32005 and data.impl_file "auth/generate_token.opq"
    And "auth/generate_token.opq" contains A's implementation
    When B's client sends submit_impl for "Auth::generate_token" again
    Then B replies with status "accepted"

  Scenario: MCP tools/call wraps the result
    When the client sends initialize with protocolVersion "2025-03-26"
    Then the result protocolVersion is "2025-03-26"
    When the client sends tools/call with name "cprime_check_impl" and arguments {"spec":"Auth::generate_token","source":"<§4.2 source>"}
    Then the result content[0].type is "text"
    And structuredContent.status is "ok" and isError is false

  Scenario: SIGTERM during submit abandons the request
    Given a spec with assurance tested whose T3 run takes 4 seconds
    When the client sends submit_impl and SIGTERM is delivered after 500 ms
    Then cprimed exits with code 0 within 200 ms of the signal
    And no reply is written

  Scenario: cprime.toml edited while running
    When the human adds a [[providers]] entry to cprime.toml
    And the client sends check_impl
    Then the reply has error code -32004

  Scenario: Happy path — witness steps carry submission spans and names
    Given verify.witness_max_steps is 64
    When the client sends submit_impl whose body is the concept §4.2 source with "(+ now 3600)" replaced by "(+ now 1000)"
    Then the reply result has status "rejected" and evidence "none"
    And the ContractViolation diagnostic has a witness with status "reproduced"
    And its param step has node 2, name "now" and span {"file": "<submission>", "line": 1, "column": 33}
    And its compute step has node 6, name "exp" and span {"file": "<submission>", "line": 2, "column": 13}
    And its return step has node 19, no name and span {"file": "<submission>", "line": 5, "column": 5}
    And timings_us has keys sync, t0, t1, t2 and witness and not t3

  Scenario: Edge case — accepted reply is unchanged by witness support
    When the client sends submit_impl for "Auth::generate_token" with the concept §4.2 source
    Then the reply result has status "accepted"
    And the result's members are exactly status, evidence, assumed, effective, impl_file, timings_us and diagnostics
    And diagnostics is empty
    And timings_us has keys sync, t0, t1, t2 and no witness key

  Scenario: Edge case — T0 rejection carries no witness and no witness timing
    When the client sends submit_impl whose body calls an undefined name "u64_le"
    Then the reply result has status "rejected" and evidence "none"
    And the only diagnostic has kind "UnknownName" and no witness member
    And timings_us has keys sync and t0 only

  Scenario: Edge case — the vacuity check runs once per spec revision (RFC 07 criterion 4)
    Given spec Auth::generate_token from concept §4.1 and a solver-call counter in a test build
    When the client sends submit_impl with the corrected source twice
    Then the first verification solves the vacuity scripts and the second solves none of them
    And both replies are accepted with evidence "proved"
    When the generate_token contract's ensures is edited on disk and the source is submitted again
    Then the vacuity scripts are solved again

  Scenario: Happy path — effective evidence names a tested callee (RFC 07 criteria 5, 6)
    Given Lib::b accepted with evidence "tested" and assumed []
    And Lib::a with assurance proved whose implementation calls Lib::b and performs CryptoCap::hmac_sha256
    When the client sends submit_impl for Lib::a and it is accepted with evidence "proved"
    Then the result member effective is {"own": "proved", "effective": "tested", "tested": ["Lib::b"], "unaccepted": [], "assumed": ["Lib::CryptoCap::hmac_sha256"]}
    When Lib::b is resubmitted and accepted with evidence "proved"
    Then assurance::effective for Lib::a returns effective "proved" and tested [] without Lib::a being resubmitted

  Scenario: Edge case — a caller accepted before its callee is implemented
    Given Lib::c has no implementation
    When the client sends submit_impl for Lib::a2, whose implementation calls Lib::c, and it is accepted with evidence "proved"
    Then effective is "none" and unaccepted is [{"spec": "Lib::c", "status": "absent"}]

  Scenario: Error path — unsupported construct is a daemon failure with its own reason (RFC 07 criterion 14)
    Given a test build whose verifier returns VerifyError::UnsupportedConstruct
    When the client sends submit_impl
    Then the reply is a JSON-RPC error -32001 with data.reason "unsupported_construct"

  Scenario: Error path — check-providers reports a failing vector
    Given a runtime built with a test provider whose deterministic operation declares a vector it does not meet
    When "cprimed --check-providers" runs
    Then it exits 1 and stderr names the provider, the operation and the vector index
```

#### Performance, Security, Observability

- **Performance targets:** Request overhead outside the tiers (JSON parse, dispatch, serialisation) ≤ 1 ms p95 for a 64 KiB `source`. `sync` ≤ 2 ms p95 for a project of ≤ 500 source files with no change (one `stat` per file). T0 ≤ 5 ms and T1 ≤ 20 ms p95 for implementations ≤ 200 IR instructions with a warm database are met by the called crates; the daemon adds no work inside those windows beyond diagnostic sorting (O(n log n), n ≤ 1,000). Persistence ≤ 5 ms p95 on a local SSD (two `fsync`), excluding time spent waiting for the project write lock (bounded at 5,000 ms). Memory: the Salsa cache is unbounded in v1; resident memory for a 500-file project is expected below 512 MiB (measured in Phase 3).
- **Security:** No socket is opened; the only client is the parent program on stdio (SA-04, A-1). No authentication. `spec` names are validated against the identifier pattern before any path is built, and `paths::impl_file` only joins identifier segments, so writes cannot leave the project root. Writes happen only in `submit_impl` after acceptance and only to `*.opq` paths and `.cprime/lock`. `--check-providers` calls provider code only for `deterministic` operations with declared vectors, in a sandbox context (D40). Submitted source and counterexamples are logged only at `trace` (SA-26). Lines are capped at 16 MiB. Symbolic links are not followed during sync.
- **Observability:** Logs only (A-9). Per-request `timings_us` in every `check_impl`/`submit_impl` reply, including `witness` when `cprime-verify` built a witness trace or ran a concrete replay (D43). The vacuity cache (D38) is per process and never written to disk. Witness traces are diagnostics returned to the client, not metrics; their step values reach the log only at `trace` (SA-26). Log targets `cprimed::start`, `cprimed::rpc`, `cprimed::sync`, `cprimed::pipeline`, `cprimed::persist`; fields `ts`, `level`, `target`, `msg`, `request_id`, `method`, `spec`, `status`, `timings_us`, `diagnostics`, `path`, `evidence`. No metrics endpoint and no tracing of `cprimed` itself (SA-33).

#### Gaps

None.
