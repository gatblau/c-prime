# Component spec — cprime-core

> **Spec set:** cprime
> **Status:** Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> **Last updated:** 2025-12-20

### SPEC: cprime-core
**File:** `crates/cprime-core/src/{lib,ids,ast,tast,ir,ir_verify,diag,value,evidence,accept,config,prelude,db,namespace,resolve,context,paths,program,hash}.rs` | **Package:** `cprime-core` | **Phase:** 1 (ids, ast, tast, diag, value, evidence, accept, config, prelude, db, namespace, resolve, context, paths, hash), 2 (ir, ir_verify, program) | **Dependencies:** none in the workspace; third-party `salsa`, `blake3`, `serde`, `serde_json`, `toml`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprime-core` is the shared foundation every other C′ crate builds on. It defines the data every stage exchanges — identifiers, the untyped and typed syntax trees, C′-IR, diagnostics, values and their JSON encoding, evidence, the acceptance header and the project configuration — and it owns the project-wide queries that need no type checking: content hashing of definitions into `DefId`s, the namespace table, name lookup, the context query behind `get_context`, path mapping between modules and files, and the IR verifier that guards every IR transformation. It holds no persistent state; its Salsa database trait is the cache through which `cprimed` and `cprimec` share these results.

#### Approach

All definitions are pure data plus pure functions over a Salsa database trait (`Db`) whose concrete implementation lives in `cprime-daemon`; three hook methods on the trait (`parse_module`, `accepted_impl`, `impl_status`) let core-owned queries reach parsing and type checking without a dependency cycle. Hashing resolves names to qualified names first, then groups definitions into strongly connected components so that mutually recursive types hash deterministically. Rejected: computing `DefId`s inside `cprime-syntax` (hashing needs the whole namespace, which a parser of one file does not see); a separate resolved-AST type (lookup is offered as a function that `cprime-types` calls while checking, so one AST type suffices).

#### Shared Context

This crate owns every type in Phase 2 §2C except those under `cprime_provider`, plus the §2E additions `NodeIndex` (D26), `TerminationInfo`, `CallEdge`, `DefKind`, `NamespaceTable`, `NsEntry`, `NsKind`, `Scope`, `Res`, `PreludeItem`, `BuiltinType`, `ContextResult`, `ContextDef`, `ContextError`. Their normative definitions are reproduced in *Public Interface* below.

Configuration keys read by this crate (all from `<root>/cprime.toml`, overridable by `--config <key>=<value>`):

| Key | Type | Default | Required | Range / format |
|---|---|---|---|---|
| `project.name` | string | — | yes | `[a-z][a-z0-9_]{0,63}` |
| `providers[].crate` | string | — | yes, per entry | non-empty, `[A-Za-z0-9_-]{1,64}` |
| `providers[].version` | string | — | exactly one of `version` / `path` | non-empty |
| `providers[].path` | path relative to root | — | exactly one of `version` / `path` | must not be absolute |
| `providers[].provider` | string | — | yes, per entry | `Ident(::Ident)*`, `Ident = [A-Za-z][A-Za-z0-9_]*`; unique across entries and the built-ins `host::Crypto`, `host::Clock`, `host::Random`, `host::Log` |
| `limits.cpu_time_ms` | u64 | 10000 | no | 1 … 86,400,000 |
| `limits.memory_bytes` | u64 | 1073741824 | no | 1,048,576 … 2^40 |
| `limits.max_call_depth` | u64 | 100000 | no | 1,000 … 10,000,000 (D6) |
| `limits.workers` | u32 | 0 | no | 0 … 256; 0 = available parallelism, capped at 256 (D20) |
| `limits.request_fuel` | u64 | 100000000 | no | 1 … 2^63; server tasks only (ADR-051) |
| `limits.shutdown_grace_ms` | u64 | 10000 | no | 0 … 600,000; servers only (ADR-051) |
| `verify.obligation_timeout_ms` | u64 | 250 | no | 1 … 600,000 |
| `verify.submission_budget_ms` | u64 | 2000 | no | 1 … 3,600,000 and ≥ `verify.obligation_timeout_ms` |
| `verify.solver_threads` | u32 | none (consumer computes min(4, available parallelism)) | no | 1 … 64 |
| `fuzz.iterations` | u64 | 10000 | no | 1 … 10,000,000 |
| `fuzz.time_budget_ms` | u64 | 5000 | no | 1 … 3,600,000 |
| `fuzz.call_fuel` | u64 | 10000000 | no | 1,000 … 2^63 |
| `fuzz.call_memory_bytes` | u64 | 67108864 | no | 65,536 … 2^40 |
| `fuzz.seed` | u64 | none (consumer derives from implementation `DefId`) | no | any |
| `verify.witness_max_steps` | u32 | 64 | no | 0 … 1024; 0 = no witness traces (D25, D29) |
| `verify.provider_wait_ms` | u64 | 2000 | no | 10 … 600,000; how long checking waits for provider code beyond a deadline (D41) |
| `release.call_chain` | bool | `false` | no | `true` \| `false` (D28, D29) |
| `log.level` | enum | `info` | no | `error` \| `warn` \| `info` \| `debug` \| `trace` |
| `runtime.cprime_source` | path | none | no | any path |

Prelude (SA-23), with the exact `ContextDef.source` line used by `get_context`:

| `PreludeItem` | Name | `source` line |
|---|---|---|
| `Hash` | `Hash` | `type Hash = FixedBytes[32]` |
| `Result` | `Result` | `enum Result[T, E] { Ok { value T }; Err { error E } }` |
| `Option` | `Option` | `enum Option[T] { Some { value T }; None }` |
| `Len` | `len` | `func len(Bytes) U64; func len(FixedBytes[N]) U64; func len(Vec[T]) U64` |
| `Index` | `index` | `func index(Vec[T], U64) T; func index(Bytes, U64) U8; func index(FixedBytes[N], U64) U8` |
| `Concat` | `++` | `func ++(Bytes, Bytes) Bytes` |
| `U64Be` | `u64_be` | `func u64_be(U64) Bytes` |
| `Push` | `push` | `func push(Vec[T], T) Vec[T]` |
| `Slice` | `slice` | `func slice(Bytes, U64, U64) Bytes` |
| `ToBytes` | `to_bytes` | `func to_bytes(FixedBytes[N]) Bytes` |
| `U8Max` | `U8::MAX` | `const U8::MAX U8 = 255` |
| `U64Max` | `U64::MAX` | `const U64::MAX U64 = 18446744073709551615` |
| `I64Min` | `I64::MIN` | `const I64::MIN I64 = -9223372036854775808` |
| `I64Max` | `I64::MAX` | `const I64::MAX I64 = 9223372036854775807` |
| `Diverge` | `Diverge` | `capability Diverge {}` |

Prelude variants: `Ok` (tag 0) and `Err` (tag 1) of `Result`; `Some` (tag 0) and `None` (tag 1) of `Option`. Builtin type names: `Unit`, `Bool`, `U8`, `U64`, `I64`, `Bytes`, `FixedBytes`, `Vec`.

#### Public Interface

Type definitions are copied verbatim from Phase 2 §2C and §2E; the module path is given in each heading. Items marked *(core-internal public)* are public for tests and not used by other crates.

##### `ids`, `ast`, `tast`, `ir`, `diag`, `value`, `evidence`, `accept`, `config`, `db`, `program`

These modules contain exactly the definitions of Phase 2 §2C sections *Identifiers*, *Untyped AST*, *Typed AST*, *C′-IR*, *Diagnostics*, *Values and Value JSON encoding* (including `TypeSchema`), *Evidence and assurance*, *Acceptance header and implementation status*, *Project configuration and manifest*, *Salsa database and inputs*, *Program bundle*, and §2E `TerminationInfo` / `CallEdge` (module `tast`). They are reproduced in Appendix A of this file without change. `Value`, `Diagnostic`, `Counterexample` serialise as specified there; `Counterexample` uses a hand-written `serde::Serialize` that writes each `Vec<(String, Value)>` as a JSON object in vector order, with values through `value::to_json`. `TraceStep` (D22) also uses a hand-written `serde::Serialize`: members in declaration order, `span`, `name` and `operands` omitted when `None`, `value` and each operand written through `value::to_json`; `WitnessStatus`, `StepKind` and `StepSource` serialise as lower-case snake case (`trap_first`, `not_reproduced`, `par_map`).

Additional items in these modules:

```rust
// db
pub struct Interner { /* Mutex-guarded maps; see Internal Logic step 10 */ }
impl Interner {
    pub fn new() -> Interner;                        // pre-registers "Diverge" as CapId(0)
    pub fn intern(&self, name: &str) -> Symbol;
    pub fn name(&self, sym: Symbol) -> Arc<str>;
    pub fn cap_id(&self, qualified: &str) -> CapId;  // "Diverge" -> CapId(0)
    pub fn cap_name(&self, cap: CapId) -> Arc<str>;
}

// ir_verify
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum IrStage { Lowered, RcInserted }
#[derive(Clone)]
pub struct IrVerifyError { pub function: DefId, pub block: u32, pub inst: Option<u32>, pub rule: &'static str, pub message: String }
pub fn verify_function(f: &ir::Function, layouts: &Layouts, stage: IrStage) -> Result<(), IrVerifyError>;

// ast — §2E (D26)
/// Source span and binder name per node id of one parsed implementation.
pub struct NodeIndex {
    pub spans: Vec<Span>,                    // spans[n] = span of the node numbered n; len == imp.node_count
    pub names: BTreeMap<NodeId, String>,     // parameter node → parameter name; let-bound value node → binder name
}
/// Cross-crate (§2E): cprime-daemon (witness step spans and names), cprime-cli (trap site and function tables).
pub fn node_index(imp: &ast::Impl) -> NodeIndex;

// config
impl std::fmt::Display for ConfigError { /* messages in the Error Table */ }
pub const BUILTIN_PROVIDERS: [&str; 4] = ["host::Crypto", "host::Clock", "host::Random", "host::Log"];
```

##### `namespace`, `resolve`, `context`, `paths`, `hash`, `accept`

```rust
// namespace — §2E
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")]
pub enum DefKind { Type, Capability, Spec, Entry, Prelude }
pub struct NamespaceTable { pub entries: BTreeMap<String, NsEntry> }
pub struct NsEntry { pub def: DefId, pub kind: NsKind, pub file: String, pub span: Span }
pub enum NsKind { Type, Capability, Spec, Entry, Server, Variant { sum: DefId, tag: u32 } }
pub fn namespace_table(db: &dyn Db) -> Arc<NamespaceTable>;                          // Salsa-tracked
pub fn namespace_diagnostics(db: &dyn Db) -> Arc<Vec<Diagnostic>>;                    // cross-crate (§2E, D11): cprime-daemon, cprime-cli
pub fn item_of(db: &dyn Db, def: DefId) -> Option<(String /* file */, usize /* item index */)>;  // core-internal public

// resolve — §2E
pub struct Scope<'a> { pub module: &'a str, pub locals: &'a [Symbol] }
pub enum Res {
    Local(u32),
    Def { def: DefId, kind: DefKind },
    Variant { sum: DefId, tag: u32 },
    CapOp { cap: CapId, cap_def: DefId, op: Symbol },
    Prelude(PreludeItem),
    Builtin(BuiltinType),
}
pub enum PreludeItem { Hash, Result, Option, Len, Index, Concat, U64Be, Push, Slice, ToBytes, U8Max, U64Max, I64Min, I64Max, Diverge }
pub enum BuiltinType { Unit, Bool, U8, U64, I64, Bytes, FixedBytes, Vec }
pub fn lookup(db: &dyn Db, scope: &Scope, path: &ast::Path) -> Result<Res, Diagnostic>;
pub fn prelude_def_id(item: PreludeItem) -> DefId;     // fixed DefIds of prelude items, Internal Logic step 3

// context — §2E
#[derive(serde::Serialize)]
pub struct ContextResult { pub spec: String, pub impl_file: String, pub implementation: ImplStatus, pub signature: String, pub definitions: Vec<ContextDef> }
#[derive(serde::Serialize)]
pub struct ContextDef { pub name: String, pub kind: DefKind, pub source: String }
pub enum ContextError { UnknownSpec { name: String }, ModuleHasErrors { file: String, diagnostics: Vec<Diagnostic> } }
pub fn get_context(db: &dyn Db, spec_name: &str) -> Result<ContextResult, ContextError>;

// paths — §2E
pub fn module_file(module: &str) -> String;           // "Auth::Session" -> "auth/session.cprime"
pub fn impl_file(spec_qualified: &str) -> String;     // "Auth::generate_token" -> "auth/generate_token.opq"
pub fn snake_case(segment: &str) -> String;           // cross-crate (§2E, D1): cprime-syntax, paths::module_file, paths::impl_file

// hash
pub const DEF_TAG: &[u8] = b"cprime.def.v1\0";
pub const CTX_TAG: &[u8] = b"cprime.ctx.v1\0";
pub fn impl_def_id(db: &dyn Db, imp: &ast::Impl) -> Result<DefId, Diagnostic>;      // cross-crate (§2E, D10): cprime-daemon, cprime-cli; cprime-verify receives the result as VerifyInput.impl_def

// accept — §2C, §2E
pub fn format_header(h: &AcceptanceHeader) -> String;
pub fn parse_header(first_line: &str) -> Result<AcceptanceHeader, HeaderError>;
pub fn context_hash(db: &dyn Db, spec: DefId, deps: &BTreeSet<DefId>) -> [u8; 32];

// value — §2C
pub fn to_json(v: &Value) -> serde_json::Value;
pub fn from_json(ty: &Type, schema: &TypeSchema, j: &serde_json::Value) -> Result<Value, ValueDecodeError>;

// config — §2C
pub fn load_config(root: &Path, overrides: &[(String, String)]) -> Result<ProjectConfig, ConfigError>;
```

##### Example

Given `auth.cprime` (concept §4.1) as the only source file:

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

- `paths::module_file("Auth")` → `"auth.cprime"`; `paths::impl_file("Auth::generate_token")` → `"auth/generate_token.opq"`; `paths::snake_case("HTTPServer")` → `"http_server"`.
- `namespace_table` contains keys `Auth::CryptoCap` (Capability), `Auth::SessionToken` (Type), `Auth::generate_token` (Spec).
- `lookup(db, &Scope { module: "Auth", locals: &[] }, CryptoCap::hmac_sha256)` → `Res::CapOp { cap: CapId(1), cap_def: <DefId of Auth::CryptoCap>, op: <Symbol "hmac_sha256"> }`.
- `get_context(db, "Auth::generate_token")` serialised by `cprime-daemon`:

```json
{
  "spec": "Auth::generate_token",
  "impl_file": "auth/generate_token.opq",
  "implementation": { "status": "absent" },
  "signature": "spec generate_token(user_id, now U64, secret Bytes) SessionToken {\n    uses CryptoCap\n    requires user_id > 0\n    requires now <= U64::MAX - 3600\n    requires len(secret) >= 32\n    ensures result.user_id == user_id\n    ensures result.expires_at == now + 3600\n    ensures result.sig == CryptoCap::hmac_sha256(secret, u64_be(user_id) ++ u64_be(result.expires_at))\n}",
  "definitions": [
    { "name": "Auth::CryptoCap", "kind": "capability", "source": "capability CryptoCap {\n    deterministic hmac_sha256(key, msg Bytes) Hash\n}" },
    { "name": "Auth::SessionToken", "kind": "type", "source": "struct SessionToken {\n    user_id    U64\n    expires_at U64\n    sig        Hash\n}" },
    { "name": "++", "kind": "prelude", "source": "func ++(Bytes, Bytes) Bytes" },
    { "name": "Hash", "kind": "prelude", "source": "type Hash = FixedBytes[32]" },
    { "name": "U64::MAX", "kind": "prelude", "source": "const U64::MAX U64 = 18446744073709551615" },
    { "name": "len", "kind": "prelude", "source": "func len(Bytes) U64; func len(FixedBytes[N]) U64; func len(Vec[T]) U64" },
    { "name": "u64_be", "kind": "prelude", "source": "func u64_be(U64) Bytes" }
  ]
}
```

- `format_header(&AcceptanceHeader { context: [0x9f, …, 0xe1], evidence: Evidence::Proved, assumed: vec!["Auth::CryptoCap::hmac_sha256".into()] })` → `";; cprime-accepted v1 context=9f…e1 evidence=proved assumed=Auth::CryptoCap::hmac_sha256\n"` (64 hex characters in full).
- `to_json(&Value::Variant { variant: "Ok".into(), fields: vec![("value".into(), Value::U64(3))] })` → `{"Ok":{"value":3}}`; `to_json(&Value::Bytes(vec![0xde, 0xad]))` → `"0xdead"`.

#### Internal Logic

1. **Interner and inputs (`db`).** `Interner` holds `Mutex<(HashMap<Arc<str>, u32>, Vec<Arc<str>>)>` for symbols and a second pair for capabilities. `Interner::new` registers `"Diverge"` as `CapId(0)`. `intern`/`cap_id` return the existing id or append. `Symbol` and `CapId` numbers depend on interning order and never enter a hash, a header or any JSON output; every hash and output uses the string. Errors: none. Logged: nothing (this crate never logs; callers log).

2. **Module/file mapping (`paths`).** `snake_case(s)`: for each character `c` at position `i > 0`, emit `_` before `c` when `c` is upper-case and (the previous character is lower-case or a digit, or the previous character is upper-case and the next character exists and is lower-case); then lower-case every character. `HTTPServer` → `http_server`, `V2Api` → `v2_api`, `Auth` → `auth`. `module_file("A::B")` = `snake_case(A) + "/" + snake_case(B) + ".cprime"`. `impl_file("A::B::spec")` = `snake_case(A) + "/" + snake_case(B) + "/" + spec + ".opq"` (the last segment is copied unchanged). Callers pass names already validated by `cprime-syntax`; the functions never panic and always return a relative path without `..`.

3. **Prelude identities (`prelude`, `resolve::prelude_def_id`).** Each `PreludeItem` has a fixed `DefId` = `blake3(DEF_TAG ++ b"prelude\0" ++ <name bytes>)`, with names from the Shared Context table (`"++"` for `Concat`). Prelude variants resolve to `Res::Variant { sum: prelude_def_id(Result|Option), tag }`.

4. **Namespace collection (`namespace::namespace_table`, step a).** Salsa-tracked over `ProjectInputs`. For every `SourceFile` whose path ends in `.cprime`, call `db.parse_module(file)`. Skip files whose `ParsedModule.module` is `None`. A module whose path disagrees with its file never reaches this step: `cprime-syntax` rejects it with `ParseError` and `ParsedModule.module` is `None` (D1). For each item except `Bind`, form the qualified name `<M>::<item name>`; for each variant of a sum type also form `<M>::<variant>`. A qualified name already present (files visited in sorted path order, items in source order) pushes `UnknownName` with message ``duplicate definition `<name>`; first defined at <file>:<line>:<column>`` and the later definition is dropped. A module-level name equal to a prelude or builtin name (for example `type Result = …`) is permitted and shadows the prelude inside that module only (lookup order, step 7). A `server` item is collected as `<M>::<name>` with `NsKind::Server` (D31).

5. **Reference graph and canonical form (`hash`, step b).** For each collected definition, resolve every path it contains to a *target*: another collected qualified name, a prelude item, a builtin type, a generic parameter (index in the type's generic list) or a local (de Bruijn index). Resolution uses the lookup order of step 7 with qualified names instead of `DefId`s. An unresolvable path is recorded as the target `Unresolved(<path text>)`, encoded as such, and reported later by `cprime-types` (namespace construction does not fail on it). Build a directed graph over collected definitions; compute strongly connected components with Tarjan's algorithm; hash components in reverse topological order so every external reference already has a `DefId`.

6. **Canonical serialisation bytes (`hash`, step c).** All integers little-endian. `str(s)` = `u32 len` + UTF-8 bytes. `list(xs)` = `u32 count` + items. A reference is one of: `0x01` + 32-byte `DefId` (external), `0x02` + `u32` member position (same component), `0x03` + `str(prelude or builtin name)`, `0x04` + `u32` generic index, `0x05` + `u32` de Bruijn index, `0x06` + `str(path text)` (unresolved). Definition bodies:
   - Type: `0x10`, `list(generic names)` replaced by `u32 count` only, then body: alias `0x11` + type-expr; record `0x12` + `list(str(field name) + type-expr)`; sum `0x13` + `list(str(variant name) + list(str(field) + type-expr))`.
   - Type-expr: `0x20` + reference + `list(arg)`, arg = `0x21` + type-expr or `0x22` + `u32` nat.
   - Capability: `0x30` + `list(u8 deterministic + str(op name) + list(type-expr) + type-expr)` (parameter names excluded).
   - Spec: `0x40` + `list(type-expr params)` + type-expr ret + `list(reference uses, sorted by encoded bytes)` + `u8 assurance (0 proved, 1 tested)` + `list(waiver)` + `list(contract-expr requires)` + `list(contract-expr ensures)`. Parameter names and the `ensures` binder name are excluded; parameters are de Bruijn locals with the binder as index 0 inside `ensures`. Waiver: `0x41` + `str(field)`, `0x42` + `str(variant)` + `str(field)`, `0x43`.
   - Entry: `0x50` + params + ret + uses, as Spec.
   - Server (D31, SA-70): `0x51` + handler reference + `str(transport path text)`.
   - Contract-expr: one tag byte per `ContractExprKind` (`0x60` Int + `i128`, `0x61` Bool + `u8`, `0x62` Bytes + `list(u8)`, `0x63` Unit, `0x64` Path + reference, `0x65` Call + reference + `list(expr)`, `0x66` Field + expr + `str(name)`, `0x67` Is + expr + `str(variant)`, `0x68` Unary + `u8 op` + expr, `0x69` Binary + `u8 op` + expr + expr). Operator codes are enum declaration order starting at 0. `ContractExpr.text` and every `Span` are excluded.
   - Implementation (`impl_def_id`): `0x70` + spec reference + `u32 param count` + `u8 has_decreases` + `list(expr measure)` + expr body. Expr tags `0x80`–`0x8C` follow `ExprKind` declaration order (`0x8C` `ParMap` + spec reference + expr, added by RFC 02 under SA-50); `Let` encodes each binding as one nested binder; `Match` arms encode variant reference + `u32 binder count` + body. Local names, the acceptance header, comments and spans are excluded.
   Component hashing: order members by their bytes encoded with every intra-component reference written as `0x02 FFFFFFFF`; ties broken by qualified name. Re-encode with real member positions; group hash = `blake3(DEF_TAG ++ list(member bytes))`; member `DefId` = `blake3(DEF_TAG ++ group hash ++ u32 position)`. A component of one non-self-referencing definition uses `DefId = blake3(DEF_TAG ++ bytes)`. Variant entries share their sum's `DefId` and carry their tag.
   **Additive layout (SA-50).** The tag bytes and layouts above are frozen. A later construct gets a new, unused tag byte; no existing tag gains, loses or reorders a field. Attributes that do not change a definition's meaning (spans, the names excluded above, and post-v1 visibility) are never encoded. `DEF_TAG` stays `cprime.def.v1`.

7. **Lookup (`resolve::lookup`).** Single-segment path `x`: (i) the innermost `locals` entry equal to `x` → `Res::Local(len - 1 - position)`; (ii) `<scope.module>::x` in the table → `Def` or `Variant`; (iii) prelude name or prelude variant → `Prelude`/`Variant`; builtin type name → `Builtin`; (iv) top-level `x` in the table → `Def`/`Variant`. Multi-segment path `p::q`: (i) `<scope.module>::p::q` in the table; (ii) the prefix `p` resolves (relative, then absolute) to a capability whose op list contains `q` → `CapOp` with `cap = interner.cap_id(<qualified capability>)` and `op = interner.intern(q)`; (iii) `U8::MAX`, `U64::MAX`, `I64::MIN`, `I64::MAX` → `Prelude`; (iv) absolute `p::q` in the table. Failure returns `Diagnostic { kind: UnknownName, tier: T0, severity: Error, node: NodeId(0), span: Some(path.span), message, .. }` where `message` is ``unknown name `<path>` in module <module>``, or ``capability `<cap>` has no operation `<op>` `` when the prefix resolved to a capability. The caller replaces `node`. `Diverge` resolves to `Res::Prelude(PreludeItem::Diverge)`; its `CapId` is `CapId(0)`.

8. **Context query (`context::get_context`).** (a) Look up `spec_name` as an absolute qualified name; absent or not `NsKind::Spec`/`Entry` → `ContextError::UnknownSpec`. (b) If the spec's file has parse diagnostics (`parse_module`) or any `namespace_diagnostics` entry for that file → `ContextError::ModuleHasErrors { file, diagnostics }`. (c) Walk from the spec: parameter and return types, every type reachable through alias targets, record fields, variant fields and generic arguments; every capability in `uses` together with the types of its operations; every capability whose operation is called in `requires`/`ensures`; every prelude item referenced by any visited definition or contract. Builtin types are not listed; the spec itself is not listed. (d) `source` of a user definition = the text from the start of its item's first line up to and including the last line before the next item's first line (or end of file) that is neither blank nor a comment-only line; `signature` uses the same rule for the spec. (e) Sort user definitions by qualified name (byte order), then prelude items by name (byte order). (f) `implementation = db.impl_status(def)`, `impl_file = paths::impl_file(spec_name)`.

9. **Acceptance header (`accept`).** `format_header` writes `;; cprime-accepted v1 context=<hex> evidence=<proved|tested> assumed=<list>\n` with `assumed` sorted, deduplicated, joined by `,`, or `-` when empty. `parse_header(line)` strips one trailing `\n` or `\r\n`, then: does not start with `;; cprime-accepted ` → `HeaderError::Missing`; otherwise split the remainder on single spaces into exactly four tokens `v1`, `context=…`, `evidence=…`, `assumed=…` in that order, else `Malformed { reason: "token layout" }`. Version other than `v1` → `"unsupported version"`; context not exactly 64 hex digits → `"context hash"` (upper-case hex accepted, normalised on format); evidence other than `proved`/`tested` → `"evidence"`; assumed entries not matching `[A-Za-z][A-Za-z0-9_]*(::[A-Za-z][A-Za-z0-9_]*)+` → `"assumed list"`. `context_hash(db, spec, deps)` = `blake3(CTX_TAG ++ concat(sorted unique {spec} ∪ deps, 32 bytes each))`; `db` is used only to keep the signature uniform with other queries.

10. **Value JSON (`value`).** `to_json` per SA-17; `Str(s)` → JSON string. `from_json(ty, schema, j)` walks `ty` and `j` together, tracking a JSONPath-style `path` starting at `$`: `Unit` requires `null`; `Bool` boolean; `U8`/`U64`/`I64` a JSON integer within range (floats and out-of-range integers fail); `Bytes` a string `0x` + even-length hex (either case); `FixedBytes(n)` the same with exactly `n` bytes; `Vec(t)` an array; `Struct(d)` an object with exactly the field names of `schema.structs[d]`; `Sum(d)` an object with exactly one key naming a variant of `schema.sums[d]` whose value is an object with exactly that variant's fields; `Fn` and `Token` always fail. Nesting deeper than 128 levels fails at the 129th. Failure → `ValueDecodeError { path, expected }`, `expected` one of: `"null"`, `"boolean"`, `"U8 integer 0..=255"`, `"U64 integer 0..=18446744073709551615"`, `"I64 integer -9223372036854775808..=9223372036854775807"`, `"hex string 0x…"`, `"hex string of <n> bytes"`, `"array"`, `"object with fields <a, b, c>"`, `"object with one variant key of <A, B>"`, `"nesting depth at most 128"`, `"a JSON-representable type"`.

11. **Configuration (`config::load_config`).** Read `<root>/cprime.toml`: missing → `ConfigError::NotFound`; larger than 1,048,576 bytes or unreadable → `Io`; invalid TOML or unknown key (serde `deny_unknown_fields` on every section) → `Toml { line, column, message }` (line/column 1-based, 0 when the parser gives no position). Apply defaults for absent sections and keys. Apply each override in order: `key` must be one of the scalar keys in the Shared Context table except `project.name` and `providers[]` keys, else `UnknownOverride`; the value is parsed as that key's type (decimal u64/u32 without sign or separators; `log.level` lower-case name; `release.call_chain` exactly `true` or `false`), else `Invalid { key, message: "expected <type>" }`. Then validate: every range and format in the Shared Context table (including `verify.witness_max_steps` 0 … 1024, message `must be between 0 and 1024`, and `verify.provider_wait_ms` 10 … 600,000, message `must be between 10 and 600000`), `submission_budget_ms >= obligation_timeout_ms`, exactly one of `version`/`path` per provider, provider paths unique and not built-in. The first failure is returned as `Invalid { key, message }`, with `key` like `providers[2].provider`.

12. **IR verifier (`ir_verify::verify_function`).** Runs the rules below in order and returns the first violation as `IrVerifyError` (`block` = block index, `inst` = instruction index within the block or `None` for params/terminator). A violation is a compiler bug (A-17); callers convert it to a JSON-RPC daemon failure, never to a rejection. *Heap types* are `Bytes`, `FixedBytes`, `Vec`, `Struct`, `Sum`. Calls and performs have result type *opaque* (no signature is available to this function); an opaque value satisfies any operand-type rule.
    - `fn-type`: `f.ty` is `Type::Fn`; `f.entry` is a valid block; entry block param types equal `params` in order.
    - `block-index`: every `BlockId` referenced is `< blocks.len()`.
    - `value-range`: every defined `ValueId` is `< f.value_count`.
    - `node-map`: every key of `f.node_map` is a value defined in `f` (D3); values without an entry are permitted.
    - `ssa-single-def`: each `ValueId` is defined exactly once (block param or instruction).
    - `reachable`: every block is reachable from `f.entry`.
    - `dominance`: every use is dominated by its definition (dominator tree by the Cooper–Harvey–Kennedy iterative algorithm; within a block, definition precedes use).
    - `edge-args`: `Jump`/`Branch`/`Switch` argument count equals the target's param count and each argument type equals the param type.
    - `branch-cond`: `Branch.cond` is `Bool`.
    - `switch-exhaustive`: `Switch.scrutinee` is `Sum(d)`, `d` is in `layouts.sums`, and `cases.len()` equals its variant count.
    - `variant-field-guard`: `GetVariantField { value, tag, .. }` lies in a block dominated by the block `cases[tag]` of a `Switch` whose scrutinee is `value`, and that case block has exactly one predecessor edge.
    - `operand-types`: `Bin` arithmetic and `Wrapping*` take two equal integer types (`U8`, `U64`, `I64`) and yield that type; `Eq`/`Ne` take two equal types other than `Fn`/`Token` and yield `Bool`; `Lt`/`Le`/`Gt`/`Ge` take equal integer types and yield `Bool`; `And`/`Or`/`Not` take `Bool`; `BytesConcat` takes `Bytes, Bytes`; `Intrinsic` per Phase 2 §2C `IntrinsicOp` comments; `MakeStruct`/`MakeVariant` field count and types match `layouts`; `GetField`/`GetVariantField` indices are in range; `MakeVec` items all equal `elem`; `Len` takes `Bytes`, `FixedBytes` or `Vec`; `Index` takes (`Vec[T]`, `U64`) → `T` or (`Bytes`|`FixedBytes`, `U64`) → `U8`.
    - `return-type`: every `Return` value has the function's return type.
    - `rc-stage`: at `IrStage::Lowered`, no `RcDup`, `RcDrop`, `DropReuse`, and every `reuse` is `None`; at `RcInserted`, operands of `RcDup`/`RcDrop`/`DropReuse` are heap-typed.
    - `token-use`: a `Token` value is produced only by `DropReuse`, used only as a `reuse` operand, and used at most once.
    - `use-after-drop`: within one block, no instruction or terminator uses a value after an `RcDrop` or `DropReuse` of that value in the same block.
    - `par-map` (D14): for `ParMap { func, xs }`, `func` is in the program and has exactly one parameter whose type equals the element type `T` of `xs : Vec(T)`, and the instruction's result type is `Vec(ret(func))`.

13. **Program bundle (`program`).** Pure data; no logic beyond construction by callers.

14. **Node index (`ast::node_index`, D26).** Allocate `spans` with `imp.node_count` entries, then make one walk in the pre-order of the `cprime-syntax` numbering rule (SA-32) and set the entry of every node it meets:
    - node 0 (`impl` form): `imp.spec.span` (the `Impl` node carries no span of its own);
    - each parameter `(Ident, NodeId)`: the `Ident.span`; also insert `names[node] = Ident.name`;
    - the `decreases` node: the span of its first measure expression (`Decreases` carries no span; a measure list is never empty);
    - every `Expr`: `Expr.span`;
    - each `Let` binding `(Ident, NodeId, Expr)`: the binder node gets `Ident.span`; insert `names[value.node] = Ident.name` for the bound value expression;
    - each `Arm`: `Arm.node` gets the span of `Arm.variant` (the variant path); each arm binder node gets its `Ident.span`.

    A node id at or above `node_count` cannot occur (`cprime-syntax` assigns `0 … node_count − 1`); the function never panics on a well-formed `Impl` and returns an empty index when `node_count` is 0. No error is possible; nothing is logged.

#### Data Model

N/A — `cprime-core` owns no database table or on-disk format beyond the acceptance header line (step 9) and `cprime.toml` (step 11). In memory it holds the Salsa-tracked `NamespaceTable` (one `BTreeMap` entry per definition and variant), the per-request diagnostics vector, and the `Interner` maps; all are rebuilt from the project files at start-up (A-7).

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Name not found by `lookup` | Diagnostic (rejected) | `UnknownName` | ``{"kind":"UnknownName","tier":"t0","severity":"error","node":0,"span":{"file":"auth/generate_token.opq","line":3,"column":12},"message":"unknown name `Crypt::hmac_sha256` in module Auth"}`` |
| Capability resolved, operation absent | Diagnostic (rejected) | `UnknownName` | ``"message":"capability `CryptoCap` has no operation `hmac_sha512`"`` |
| Two definitions with one qualified name | Diagnostic (module error) | `UnknownName` | ``"message":"duplicate definition `Auth::SessionToken`; first defined at auth.cprime:5:1"`` |
| Spec name unknown to `get_context` | `Err` | `ContextError::UnknownSpec` | `UnknownSpec { name: "Auth::make_token" }` |
| Spec's module has parse or namespace errors | `Err` | `ContextError::ModuleHasErrors` | `ModuleHasErrors { file: "auth.cprime", diagnostics: [...] }` |
| IR rule violated | `Err` (compiler bug, A-17) | `IrVerifyError` | `IrVerifyError { function: <DefId>, block: 2, inst: Some(4), rule: "dominance", message: "v17 used before its definition in block 3" }` |
| JSON value does not match type | `Err` | `ValueDecodeError` | `ValueDecodeError { path: "$.secret", expected: "hex string 0x…" }` |
| First line is not an acceptance header | `Err` | `HeaderError::Missing` | `Missing` |
| Header present but malformed | `Err` | `HeaderError::Malformed` | `Malformed { reason: "context hash" }` |
| `cprime.toml` absent | `Err` | `ConfigError::NotFound` | `cprime.toml not found in /work/auth_service` |
| `cprime.toml` unreadable or > 1 MiB | `Err` | `ConfigError::Io` | `cannot read /work/auth_service/cprime.toml: file larger than 1048576 bytes` |
| TOML syntax error or unknown key | `Err` | `ConfigError::Toml` | `cprime.toml:7:1: unknown field `budget`, expected one of `obligation_timeout_ms`, `submission_budget_ms`, `solver_threads`` |
| Value out of range or exclusivity broken | `Err` | `ConfigError::Invalid` | `invalid value for verify.submission_budget_ms: must be at least verify.obligation_timeout_ms (250)` |
| `--config` key not overridable | `Err` | `ConfigError::UnknownOverride` | `unknown configuration key in --config: verify.budget` |
| IR `ParMap` whose function has the wrong shape | `Err` (compiler bug, A-17) | `IrVerifyError` | `IrVerifyError { function: <DefId>, block: 0, inst: Some(3), rule: "par-map", message: "ParMap function takes 2 parameters; expected 1" }` |
| `limits.workers` above 256 | `Err` | `ConfigError::Invalid` | `invalid value for limits.workers: must be 0 … 256` |
| `limits.request_fuel` is 0 or above 2^63, or `limits.shutdown_grace_ms` above 600,000 | `Err` | `ConfigError::Invalid` | `invalid value for limits.request_fuel: must be 1 … 9223372036854775808` (and the matching message for `limits.shutdown_grace_ms`) |
| `verify.witness_max_steps` above 1024 | `Err` | `ConfigError::Invalid` | `invalid value for verify.witness_max_steps: must be between 0 and 1024` |
| `verify.provider_wait_ms` outside 10 … 600,000 (D41) | `Err` | `ConfigError::Invalid` | `invalid value for verify.provider_wait_ms: must be between 10 and 600000` |
| `release.call_chain` override not `true` or `false` | `Err` | `ConfigError::Invalid` | `invalid value for release.call_chain: expected bool` (in `cprime.toml`, a non-boolean value is a TOML type error → `ConfigError::Toml`) |

`ConfigError` `Display` formats, exactly: `NotFound` → `cprime.toml not found in <root>`; `Io` → `cannot read <path>: <message>`; `Toml` → `cprime.toml:<line>:<column>: <message>`; `Invalid` → `invalid value for <key>: <message>`; `UnknownOverride` → `unknown configuration key in --config: <key>`.

#### Acceptance Criteria (Gherkin)

```gherkin
Feature: cprime-core

  Scenario: Happy path — namespace and context for the concept example
    Given a project whose only source file is "auth.cprime" with the concept §4.1 content
    When namespace_table is computed
    Then it contains "Auth::CryptoCap", "Auth::SessionToken" and "Auth::generate_token"
    And get_context("Auth::generate_token") lists definitions in the order "Auth::CryptoCap", "Auth::SessionToken", "++", "Hash", "U64::MAX", "len", "u64_be"
    And its implementation status is "absent" and impl_file is "auth/generate_token.opq"

  Scenario: Edge case — renaming a spec parameter does not change its DefId
    Given "auth.cprime" with spec generate_token(user_id: U64, now: U64, secret: Bytes)
    When the parameter "now" is renamed to "t" in the signature and in every contract clause
    Then the DefId of "Auth::generate_token" is unchanged

  Scenario: Edge case — mutually recursive types hash deterministically
    Given "tree.cprime" declaring "enum Tree { Leaf; Node { kids Forest } }" and "struct Forest { items Vec[Tree] }"
    When the files are loaded twice with the two declarations in swapped source order
    Then "Tree::Tree" and "Tree::Forest" receive the same DefIds both times

  Scenario: Edge case — snake_case conversion of acronyms and digits
    When module_file("HTTPServer::V2Api") is called
    Then it returns "http_server/v2_api.cprime"

  Scenario: Error path — unknown operation on a known capability
    Given module Auth declares capability CryptoCap with operation hmac_sha256 only
    When lookup resolves "CryptoCap::hmac_sha512" in scope module "Auth"
    Then it returns a diagnostic of kind UnknownName with message "capability `CryptoCap` has no operation `hmac_sha512`"

  Scenario: Error path — duplicate definition
    Given "auth.cprime" declares "type SessionToken" on line 5 and again on line 9
    When namespace_table is computed
    Then namespace_diagnostics contains UnknownName "duplicate definition `Auth::SessionToken`; first defined at auth.cprime:5:1"
    And get_context("Auth::generate_token") returns ModuleHasErrors for "auth.cprime"

  Scenario: Error path — IR verifier rejects a Switch with too few cases
    Given a lowered function whose Switch on Result has one case
    When verify_function runs at stage Lowered
    Then it returns IrVerifyError with rule "switch-exhaustive"

  Scenario: Error path — RC instruction before RC insertion
    Given a function containing RcDup at stage Lowered
    When verify_function runs
    Then it returns IrVerifyError with rule "rc-stage"

  Scenario: Error path — IR verifier rejects a ParMap of the wrong shape
    Given a lowered function with ParMap over Vec[U64] whose function takes Bytes
    When verify_function runs at stage Lowered
    Then it returns IrVerifyError with rule "par-map"
  Scenario: Happy path — acceptance header round trip
    Given an AcceptanceHeader with evidence tested and assumed ["Auth::CryptoCap::hmac_sha256", "Auth::CryptoCap::hmac_sha256"]
    When it is formatted and parsed back
    Then the formatted line ends with "evidence=tested assumed=Auth::CryptoCap::hmac_sha256\n"
    And parse_header returns the same context, evidence tested and one assumed entry

  Scenario: Error path — malformed header
    When parse_header(";; cprime-accepted v2 context=00 evidence=proved assumed=-") is called
    Then it returns Malformed with reason "unsupported version"

  Scenario: Error path — FixedBytes length mismatch in JSON
    When from_json decodes "0xdead" against FixedBytes(32)
    Then it returns ValueDecodeError with path "$" and expected "hex string of 32 bytes"

  Scenario: Error path — configuration override for an unknown key
    Given a valid cprime.toml
    When load_config is called with override ("verify.budget", "10")
    Then it returns UnknownOverride with key "verify.budget"

  Scenario: Error path — budget smaller than per-obligation timeout
    Given cprime.toml sets verify.obligation_timeout_ms = 500 and verify.submission_budget_ms = 400
    When load_config is called
    Then it returns Invalid with key "verify.submission_budget_ms"

  Scenario: Happy path — node_index of the concept example
    Given "auth/generate_token.opq" whose first line is ";; implements Auth::generate_token" followed by the concept §4.2 implementation
    And parse_opq has returned its Impl with node_count 23
    When node_index is called
    Then spans has 23 entries
    And names maps node 1 to "uid", node 2 to "now" and node 3 to "secret"
    And names maps node 6 to "exp", node 10 to "msg" and node 16 to "sig"
    And spans[6] is line 3, column 13 of "auth/generate_token.opq"
    And names has no entry for node 5

  Scenario: Error path — witness_max_steps above its range
    Given cprime.toml sets verify.witness_max_steps = 2000
    When load_config is called
    Then it returns Invalid with key "verify.witness_max_steps" and message "must be between 0 and 1024"

  Scenario: Error path — provider_wait_ms below its range (D41)
    Given --config verify.provider_wait_ms=5
    When load_config is called
    Then it returns Invalid with key "verify.provider_wait_ms" and message "must be between 10 and 600000"
```

#### Performance, Security, Observability

- **Performance targets:** `lookup` ≤ 2 µs p95 (one `BTreeMap` probe per candidate). Warm `namespace_table` (unchanged inputs) is a Salsa cache hit, ≤ 10 µs. Cold `namespace_table` for 500 definitions across 50 files ≤ 20 ms. `get_context` ≤ 2 ms p95 for ≤ 200 reachable definitions. `verify_function` ≤ 300 µs p95 for 200 instructions (it runs inside T1's 20 ms). `impl_def_id` ≤ 200 µs for 200 nodes. `load_config` ≤ 5 ms. `node_index` ≤ 50 µs for 200 nodes (one walk, no allocation beyond the two maps).
- **Security:** The only file read is `cprime.toml` (capped at 1 MiB). `paths` functions only ever produce relative paths made of validated segments; they cannot produce `..` or an absolute path, so an agent-supplied spec name cannot direct a write outside the project. `from_json` caps nesting at 128 to bound recursion on input from release-binary standard input. No secret values pass through this crate; counterexample values are synthetic (HLD Data Lifecycle).
- **Observability:** `cprime-core` writes no logs and no metrics; it reports through returned `Diagnostic`s and error values, which `cprime-daemon` and `cprime-cli` log under the cross-cutting Logging spec.

#### Gaps

None.

---

#### Appendix A — Type definitions (verbatim from Phase 2 §2C and §2E)

```rust
// ---------- ids ----------
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct Symbol(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct DefId(pub [u8; 32]);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct CapId(pub u32);   // CapId(0) = Diverge
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, serde::Serialize)] #[serde(transparent)] pub struct NodeId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct ValueId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct BlockId(pub u32);

// ---------- ast ----------
#[derive(Clone, PartialEq, Eq)] pub struct Ident { pub name: String, pub span: Span }
#[derive(Clone, PartialEq, Eq)] pub struct Path { pub segments: Vec<Ident>, pub span: Span }
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
pub struct BindDef { pub capability: Path, pub provider: Path, pub span: Span }
pub struct ServerDef { pub name: Ident, pub handler: Path, pub transport: Path, pub span: Span }   // ADR-051; handler: spec path, transport: provider path
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

// ---------- ir ----------
#[derive(Clone, PartialEq, Eq, Hash)]
pub enum Type { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<Type>), Struct(DefId), Sum(DefId),
                Fn { params: Vec<Type>, ret: Box<Type>, effects: BTreeSet<CapId> }, Token }
#[derive(Clone, PartialEq, Eq, Hash)] pub enum Const { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(Vec<u8>) }
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub enum BinOp { Add, Sub, Mul, Div, Rem, WrappingAdd, WrappingSub, WrappingMul, Eq, Ne, Lt, Le, Gt, Ge, And, Or, BytesConcat }
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub enum IntrinsicOp { U64Be /* (U64)->Bytes */, Push /* (Vec<T>,T)->Vec<T> */, Slice /* (Bytes,U64,U64)->Bytes */, ToBytes /* (FixedBytes<N>)->Bytes */ }
#[derive(Clone)]
pub enum Inst {
    Const(Const), Bin { op: BinOp, lhs: ValueId, rhs: ValueId }, Not(ValueId),
    Call { func: DefId, args: Vec<ValueId> }, Perform { cap: CapId, op: Symbol, args: Vec<ValueId> },
    Intrinsic { op: IntrinsicOp, args: Vec<ValueId> },
    MakeStruct { ty: DefId, fields: Vec<ValueId>, reuse: Option<ValueId> }, GetField { value: ValueId, index: u32 },
    MakeVariant { ty: DefId, tag: u32, fields: Vec<ValueId>, reuse: Option<ValueId> },
    GetVariantField { value: ValueId, tag: u32, index: u32 },
    MakeVec { elem: Type, items: Vec<ValueId> }, Len(ValueId), Index { vec: ValueId, index: ValueId },
    ParMap { func: DefId, xs: ValueId },
    RcDup(ValueId), RcDrop(ValueId), DropReuse(ValueId),
}
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub enum TrapKind { Overflow, DivisionByZero, IndexOutOfBounds, Unreachable }
#[derive(Clone)]
pub enum Terminator {
    Return(ValueId), Jump { target: BlockId, args: Vec<ValueId> },
    Branch { cond: ValueId, then_target: BlockId, then_args: Vec<ValueId>, else_target: BlockId, else_args: Vec<ValueId> },
    Switch { scrutinee: ValueId, cases: Vec<(BlockId, Vec<ValueId>)> }, Trap(TrapKind),
}
#[derive(Clone)] pub struct Block { pub params: Vec<(ValueId, Type)>, pub insts: Vec<(ValueId, Inst)>, pub term: Terminator }
#[derive(Clone)] pub struct Function { pub def: DefId, pub ty: Type, pub entry: BlockId, pub blocks: Vec<Block>, pub value_count: u32,
                                       pub node_map: BTreeMap<ValueId, NodeId> }  // (D3, D23) source node of every parameter, non-constant instruction, join parameter and Return value
pub struct Layouts { pub structs: BTreeMap<DefId, Vec<Type>>, pub sums: BTreeMap<DefId, Vec<Vec<Type>>> }

// ---------- tast ----------
pub struct TSpec { pub def: DefId, pub name: String, pub params: Vec<(Symbol, Type)>, pub ret: Type, pub uses: BTreeSet<CapId>,
                   pub assurance: Assurance, pub waivers: Vec<Waiver>, pub requires: Vec<TClause>, pub ensures: Vec<TClause>, pub is_entry: bool }
pub enum Waiver { Field(Symbol), VariantField { variant: Symbol, field: Symbol }, VariantTest }
pub struct TImpl { pub spec: DefId, pub params: Vec<(Symbol, NodeId)>, pub decreases: Option<TDecreases>, pub body: TExpr, pub node_count: u32 }
pub struct TDecreases { pub node: NodeId, pub measure: Vec<TExpr> }
pub struct TExpr { pub node: NodeId, pub ty: Type, pub kind: TExprKind }
pub enum TExprKind {
    Const(Const), Local(u32), Let { value: Box<TExpr>, body: Box<TExpr> },
    If { cond: Box<TExpr>, then_: Box<TExpr>, else_: Box<TExpr> }, Bin { op: BinOp, lhs: Box<TExpr>, rhs: Box<TExpr> },
    Not(Box<TExpr>), And(Box<TExpr>, Box<TExpr>), Or(Box<TExpr>, Box<TExpr>),
    Call { func: DefId, args: Vec<TExpr> }, Perform { cap: CapId, op: Symbol, deterministic: bool, args: Vec<TExpr> },
    Intrinsic { op: IntrinsicOp, args: Vec<TExpr> }, Len(Box<TExpr>), Index { vec: Box<TExpr>, index: Box<TExpr> },
    MakeStruct { ty: DefId, fields: Vec<TExpr> }, GetField { value: Box<TExpr>, index: u32 },
    MakeVariant { ty: DefId, tag: u32, fields: Vec<TExpr> }, Match { scrutinee: Box<TExpr>, arms: Vec<TArm> },
    MakeVec { elem: Type, items: Vec<TExpr> },
    ParMap { func: DefId, xs: Box<TExpr> },
}
pub struct TArm { pub tag: u32, pub binders: u32, pub body: TExpr }
pub struct TClause { pub text: String, pub span: Span, pub expr: CExpr }
pub struct CExpr { pub ty: Type, pub kind: CExprKind }
pub enum CExprKind {
    Int(i128), Const(Const), Param(u32), Result,
    Bin { op: CBinOp, lhs: Box<CExpr>, rhs: Box<CExpr> }, Not(Box<CExpr>),
    And(Box<CExpr>, Box<CExpr>), Or(Box<CExpr>, Box<CExpr>), Implies(Box<CExpr>, Box<CExpr>),
    Is { value: Box<CExpr>, tag: u32 }, Field { value: Box<CExpr>, index: u32 }, VariantField { value: Box<CExpr>, tag: u32, index: u32 },
    Intrinsic { op: IntrinsicOp, args: Vec<CExpr> }, Len(Box<CExpr>), Index { vec: Box<CExpr>, index: Box<CExpr> },
    OpApp { cap: CapId, op: Symbol, qualified: String, args: Vec<CExpr> },
}
#[derive(Clone, Copy, PartialEq, Eq)] pub enum CBinOp { Add, Sub, Mul, Div, Rem, Eq, Ne, Lt, Le, Gt, Ge, Concat }
pub struct TerminationInfo { pub component: BTreeSet<DefId>, pub diverges: bool, pub edges: Vec<CallEdge>, pub diagnostics: Vec<Diagnostic> }
pub struct CallEdge { pub caller: DefId, pub callee: DefId, pub node: NodeId, pub caller_is_candidate: bool }

// ---------- diag ----------
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)]
pub enum DiagnosticKind { ParseError, UnknownName, ArityMismatch, TypeMismatch, UnauthorizedEffect, MissingDecreases, NonExhaustiveMatch,
                          UnconstrainedOutput, InvalidParMap, ContractViolation, ReachableTrap, NonDecreasing, VerificationInconclusive, NonDeterministicProvider, InvalidServer,
                          VacuousContract, VacuityUnknown }   // (D37)
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")] pub enum Tier { T0, T1, T2, T3 }
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")] pub enum Severity { Error, Note }   // (D37) Note only on VacuityUnknown
#[derive(Clone, PartialEq, Eq, serde::Serialize)] pub struct Span { pub file: String, pub line: u32, pub column: u32 }
#[derive(Clone, serde::Serialize)]
pub struct Diagnostic {
    pub kind: DiagnosticKind, pub tier: Tier, pub severity: Severity, pub node: NodeId,
    #[serde(skip_serializing_if = "Option::is_none")] pub span: Option<Span>,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")] pub clause: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")] pub counterexample: Option<Counterexample>,
    #[serde(skip_serializing_if = "Option::is_none")] pub reason: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")] pub witness: Option<WitnessTrace>,   // (D22)
    #[serde(skip_serializing_if = "Option::is_none")] pub concrete: Option<ConcreteCheck>, // (D43) last field
}
// (D43, ADR-055)
#[derive(Clone, serde::Serialize)] pub struct ConcreteCheck { pub status: ConcreteStatus,
    #[serde(skip_serializing_if = "Option::is_none")] pub reason: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")] pub witness: Option<WitnessTrace> }
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")] pub enum ConcreteStatus { Concrete, Abstract }
// ConcreteCheck.reason values: "no_violation", "different_outcome", "callee_unimplemented", "stubbed_operation", "resource_limit",
//   "provider_failure", "provider_timeout"
#[derive(Clone)] pub struct Counterexample { pub inputs: Vec<(String, Value)>, pub expected: Vec<(String, Value)>, pub actual: Vec<(String, Value)> }
// reason values: "solver_unknown", "timeout", "budget_cut_off", "fuzz_budget_incomplete", "fuzz_resource_limit", "callee_unimplemented",
//   "unsupported_construct" (D44), "provider_failure" (D42), "provider_timeout" (D41)
// (D22) witness trace — attached by cprime-verify to ContractViolation and ReachableTrap diagnostics that carry a counterexample
#[derive(Clone, serde::Serialize)] pub struct WitnessTrace { pub status: WitnessStatus, pub steps: Vec<TraceStep>, pub omitted: u32 }
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "snake_case")] pub enum WitnessStatus { Reproduced, TrapFirst, NotReproduced }
#[derive(Clone)] pub struct TraceStep { pub node: NodeId, pub span: Option<Span>, pub name: Option<String>, pub kind: StepKind,
    pub value: Value, pub operands: Option<Vec<Value>>, pub source: StepSource }   // hand-written Serialize (Public Interface)
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "snake_case")]
pub enum StepKind { Param, Compute, Construct, Call, Perform, ParMap, Branch, Match, Return, Trap }
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "snake_case")]
pub enum StepSource { Input, Computed, Solver, Generated, Provider }
// value per kind: Branch → Bool (the condition); Match → Str(variant name); Trap → Str(TrapKind name); otherwise the step's value.
// operands: Some only for Compute and Trap — operand values in operand order.

// ---------- value ----------
#[derive(Clone, PartialEq, Eq)]
pub enum Value { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(Vec<u8>), FixedBytes(Vec<u8>), Vec(Vec<Value>),
                 Struct { fields: Vec<(String, Value)> }, Variant { variant: String, fields: Vec<(String, Value)> }, Str(String) }
pub struct TypeSchema { pub structs: BTreeMap<DefId, Vec<(String, Type)>>, pub sums: BTreeMap<DefId, Vec<(String, Vec<(String, Type)>)>> }
pub struct ValueDecodeError { pub path: String, pub expected: String }

// ---------- evidence ----------
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")] pub enum Evidence { Proved, Tested, None }
#[derive(Clone, Copy, PartialEq, Eq)] pub enum Assurance { Proved, Tested }
pub struct VerificationResult { pub accepted: bool, pub evidence: Evidence, pub assumed: Vec<String>, pub diagnostics: Vec<Diagnostic>, pub t2_us: u64, pub t3_us: Option<u64>,
    pub witness_us: Option<u64>,    // (D25, D43) Some iff at least one witness or concrete replay ran
    pub vacuity: Option<VacuityOutcome> }   // (D37)
#[derive(Clone)] pub enum VacuityOutcome { Satisfiable, Vacuous { diagnostic: Diagnostic }, Unknown { reason: &'static str } }   // (D37, D38)
#[derive(Clone, PartialEq, Eq, serde::Serialize)]
pub struct EffectiveEvidence { pub own: Evidence, pub effective: Evidence, pub tested: Vec<String>,
    pub unaccepted: Vec<UnacceptedCallee>, pub assumed: Vec<String> }   // (D39); computed by cprime-daemon, defined here
#[derive(Clone, PartialEq, Eq, serde::Serialize)] pub struct UnacceptedCallee { pub spec: String, pub status: String }

// ---------- accept ----------
pub struct AcceptanceHeader { pub context: [u8; 32], pub evidence: Evidence, pub assumed: Vec<String> }
pub enum HeaderError { Missing, Malformed { reason: &'static str } }
#[derive(Clone, PartialEq, Eq, serde::Serialize)] #[serde(tag = "status", rename_all = "lowercase")]
pub enum ImplStatus { Absent, Current { evidence: Evidence, assumed: Vec<String> }, Stale, Invalid { reason: String } }

// ---------- config ----------
#[derive(Clone, serde::Deserialize)] #[serde(deny_unknown_fields)]
pub struct ProjectConfig { pub project: ProjectSection, #[serde(default)] pub providers: Vec<ProviderEntry>, #[serde(default)] pub limits: Limits,
    #[serde(default)] pub verify: VerifyConfig, #[serde(default)] pub fuzz: FuzzConfig, #[serde(default)] pub log: LogConfig, #[serde(default)] pub runtime: RuntimeConfig,
    #[serde(default)] pub release: ReleaseConfig }   // (D29)
pub struct ProjectSection { pub name: String }
pub struct ProviderEntry { #[serde(rename = "crate")] pub crate_name: String, pub version: Option<String>, pub path: Option<PathBuf>, pub provider: String }
pub struct Limits { pub cpu_time_ms: u64, pub memory_bytes: u64, pub max_call_depth: u64, pub workers: u32, pub request_fuel: u64, pub shutdown_grace_ms: u64 }
pub struct VerifyConfig { pub obligation_timeout_ms: u64, pub submission_budget_ms: u64, pub solver_threads: Option<u32>,
    pub witness_max_steps: u32,    // (D29) 0 … 1024, default 64
    pub provider_wait_ms: u64 }    // (D41) 10 … 600,000, default 2000
pub struct FuzzConfig { pub iterations: u64, pub time_budget_ms: u64, pub call_fuel: u64, pub call_memory_bytes: u64, pub seed: Option<u64> }
pub struct LogConfig { pub level: LogLevel }
#[derive(Clone, Copy, serde::Deserialize)] #[serde(rename_all = "lowercase")] pub enum LogLevel { Error, Warn, Info, Debug, Trace }
pub struct RuntimeConfig { pub cprime_source: Option<PathBuf> }
pub struct ReleaseConfig { pub call_chain: bool }   // (D28, D29) default false
pub enum ConfigError { NotFound { root: PathBuf }, Io { path: PathBuf, message: String }, Toml { line: u32, column: u32, message: String },
                       Invalid { key: String, message: String }, UnknownOverride { key: String } }

// ---------- db ----------
#[salsa::input] pub struct SourceFile { pub path: String, pub text: String }
#[salsa::input] pub struct ProjectInputs { pub files: Vec<SourceFile>, pub config: ProjectConfig, pub providers: Vec<ProviderInfo> }
#[derive(Clone, PartialEq, Eq, Hash)]
pub enum BoundaryTy { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<BoundaryTy>), Result(Box<BoundaryTy>, Box<BoundaryTy>) }
#[derive(Clone, PartialEq, Eq, Hash)] pub struct ProviderOpInfo { pub name: String, pub params: Vec<BoundaryTy>, pub ret: BoundaryTy, pub deterministic: bool, pub flat_index: u32,
    pub vectors: u32 }   // (D40)
#[derive(Clone, PartialEq, Eq, Hash)] pub struct ProviderInfo { pub path: String, pub crate_name: String, pub crate_version: String, pub ops: Vec<ProviderOpInfo>, pub thread_safe: bool, pub transport: Option<TransportInfo> }
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

// ---------- program ----------
pub struct Program { pub functions: BTreeMap<DefId, ir::Function>, pub layouts: Layouts, pub schema: TypeSchema,
                     pub borrows: BTreeMap<DefId, BorrowSig>, pub op_targets: BTreeMap<(CapId, Symbol), OpTarget>, pub unimplemented: BTreeSet<DefId> }
#[derive(Clone, PartialEq, Eq)] pub struct BorrowSig { pub borrowed: Vec<bool> }
#[derive(Clone, Copy, PartialEq, Eq)] pub enum OpTarget { Provider { flat_index: u32 }, Stub }
```
