# Component spec — cprime-cli

> Spec set: cprime
> Status: Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> Last updated: 2025-12-20

### SPEC: cprime-cli
**File:** `crates/cprime-cli/src/main.rs` (modules `args.rs`, `launcher.rs`, `fingerprint.rs`, `runtime_build.rs`, `release.rs`, `release_main_gen.rs`, `inventory.rs`, `status.rs`, `cargo.rs`, `describe.rs`) | **Package:** `cprime-cli` (binary `cprimec`) | **Phase:** 3 (launcher, runtime build, status), 5 (release build, inventory) | **Dependencies:** `cprime-core`, `cprime-syntax`, `cprime-types`, `cprime-effects`, `cprime-memory`, `cprime-codegen`, `cprime-provider`, `cprime-daemon`; third-party `clap`, `serde`, `serde_json`, `libc`, `blake3`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprimec` is the command-line front end. It is the launcher an agent host runs to start a project's daemon (`cprimec daemon`), the builder of the project-specific runtime (a generated Cargo package that statically links `cprime-daemon`, `cprime-provider`, the four built-in providers and the manifest's providers into the project's own `cprimed` and into `libcprime_runtime.a`), the release builder that AOT-compiles the accepted implementations through LLVM and links them into a native binary with a generated `main`, and the reporter of the provider inventory and of implementation status. This spec also defines the two generated Cargo packages: `.cprime/runtime/` and `.cprime/release/`.

#### Approach

`cprimec` is a generic binary that links no project provider. Everything that needs project providers runs inside the project's own `cprimed` (built by `cprimec runtime build`), and `cprimec` learns the provider set by running `.cprime/bin/cprimed --describe-providers` and caching the JSON in `.cprime/providers.json`. The release `main` is generated Rust code compiled by Cargo, which keeps JSON decoding out of the runtime library and avoids a separate system-linker step. Rejected alternatives: loading providers into `cprimec` with `dlopen` — dynamic loading is a non-goal; emitting `main` in LLVM IR — every Value JSON decoder would have to be written in IR.

#### Shared Context

Copied from `phase-2-architecture.md` §2C/§2D/§2E; only used members are reproduced.

```rust
// cprime_core::ids / diag (used to render diagnostics)
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct DefId(pub [u8; 32]);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct CapId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct Symbol(pub u32);
pub struct Span { pub file: String, pub line: u32, pub column: u32 }
pub struct Diagnostic { pub kind: DiagnosticKind, pub tier: Tier, pub severity: Severity, pub node: NodeId,
    pub span: Option<Span>, pub message: String, pub clause: Option<String>,
    pub counterexample: Option<Counterexample>, pub reason: Option<String>,
    pub witness: Option<WitnessTrace>,    // (D22) WitnessTrace is defined in cprime_core::diag; release re-checks run T0 only, so always None here
    pub concrete: Option<ConcreteCheck> } // (D43) always None here (T0 only)
// Severity { Error, Note } (D37); release re-checks produce Error only

// cprime_core::accept
#[derive(Clone, PartialEq, Eq, serde::Serialize)] #[serde(tag = "status", rename_all = "lowercase")]
pub enum ImplStatus { Absent, Current { evidence: Evidence, assumed: Vec<String> }, Stale, Invalid { reason: String } }
// cprime_core::evidence (D39)
#[derive(Clone, PartialEq, Eq, serde::Serialize)]
pub struct EffectiveEvidence { pub own: Evidence, pub effective: Evidence, pub tested: Vec<String>,
    pub unaccepted: Vec<UnacceptedCallee>, pub assumed: Vec<String> }
#[derive(Clone, PartialEq, Eq, serde::Serialize)]
pub struct UnacceptedCallee { pub spec: String, pub status: String }   // "absent" | "stale" | "invalid"
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")]
pub enum Evidence { Proved, Tested, None }

// cprime_core::config
pub struct ProjectConfig { pub project: ProjectSection, pub providers: Vec<ProviderEntry>, pub limits: Limits,
    pub verify: VerifyConfig, pub fuzz: FuzzConfig, pub log: LogConfig, pub runtime: RuntimeConfig,
    pub release: ReleaseConfig }                                       // (D29) [release] section
pub struct ProjectSection { pub name: String }                        // [a-z][a-z0-9_]{0,63}
pub struct ProviderEntry { pub crate_name: String, pub version: Option<String>, pub path: Option<PathBuf>, pub provider: String }
pub struct Limits { pub cpu_time_ms: u64, pub memory_bytes: u64, pub max_call_depth: u64, pub workers: u32, pub request_fuel: u64, pub shutdown_grace_ms: u64 }
pub struct RuntimeConfig { pub cprime_source: Option<PathBuf> }
pub struct ReleaseConfig { pub call_chain: bool }                     // (D28, D29) default false
pub fn load_config(root: &Path, overrides: &[(String, String)]) -> Result<ProjectConfig, ConfigError>;
pub enum ConfigError { NotFound { root: PathBuf }, Io { path: PathBuf, message: String },
    Toml { line: u32, column: u32, message: String }, Invalid { key: String, message: String }, UnknownOverride { key: String } }

// cprime_core::ir / program / value (used by the release build)
pub enum Type { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<Type>), Struct(DefId), Sum(DefId),
    Fn { params: Vec<Type>, ret: Box<Type>, effects: BTreeSet<CapId> }, Token }
pub struct Layouts { pub structs: BTreeMap<DefId, Vec<Type>>, pub sums: BTreeMap<DefId, Vec<Vec<Type>>> }
pub struct TypeSchema { pub structs: BTreeMap<DefId, Vec<(String, Type)>>, pub sums: BTreeMap<DefId, Vec<(String, Vec<(String, Type)>)>> }
pub struct Program { pub functions: BTreeMap<DefId, ir::Function>, pub layouts: Layouts, pub schema: TypeSchema,
    pub borrows: BTreeMap<DefId, BorrowSig>, pub op_targets: BTreeMap<(CapId, Symbol), OpTarget>, pub unimplemented: BTreeSet<DefId> }
pub struct BorrowSig { pub borrowed: Vec<bool> }
pub enum OpTarget { Provider { flat_index: u32 }, Stub }

// cprime_core::db (provider mirror)
pub enum BoundaryTy { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<BoundaryTy>), Result(Box<BoundaryTy>, Box<BoundaryTy>) }
pub struct ProviderOpInfo { pub name: String, pub params: Vec<BoundaryTy>, pub ret: BoundaryTy, pub deterministic: bool, pub flat_index: u32,
    pub vectors: u32 }   // (D40) from providers.json "vectors"
pub struct ProviderInfo { pub path: String, pub crate_name: String, pub crate_version: String, pub ops: Vec<ProviderOpInfo>, pub thread_safe: bool, pub transport: Option<TransportInfo> }
#[derive(Clone, PartialEq, Eq, Hash)]
pub struct TransportInfo { pub version: u32, pub request: Vec<(String, BoundaryTy)>, pub response: Vec<(String, BoundaryTy)> }   // ADR-051, SA-66

// cprime_core::namespace
pub struct NamespaceTable { pub entries: BTreeMap<String, NsEntry> }
pub struct NsEntry { pub def: DefId, pub kind: NsKind, pub file: String, pub span: Span }
pub enum NsKind { Type, Capability, Spec, Entry, Server, Variant { sum: DefId, tag: u32 } }

// cprime_core::ast (§2E, D26) — source locations for the trap site and function tables
pub struct NodeIndex { pub spans: Vec<Span>, pub names: BTreeMap<NodeId, String> }   // spans[n]: span of node n; len == imp.node_count
pub fn node_index(imp: &ast::Impl) -> NodeIndex;                                    // node 0 (impl form) takes the span of the spec path

// cprime_codegen::aot (§2E, D27, D28)
pub struct AotOptions { pub call_chain: bool, pub trap_sites: bool }
// trap_sites: cprimec always passes true; never read from configuration; false exists only for the Phase 5 baseline benchmark
pub struct AotOutput { pub object: PathBuf, pub entry_symbol: String,
    pub sites: Vec<AotSite>,          // site id = index + 1; empty when trap_sites is false
    pub functions: Vec<DefId> }       // function index = position + 1
pub struct AotSite { pub func: DefId, pub node: NodeId, pub kind: AotSiteKind }
pub enum AotSiteKind { Trap(TrapKind), Entry, Perform { cap: CapId, op: Symbol } }
pub enum TrapKind { Overflow, DivisionByZero, IndexOutOfBounds, Unreachable }

// cprime_provider
pub enum BoundaryType { Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<BoundaryType>), Result(Box<BoundaryType>, Box<BoundaryType>) }
pub type OpFn = unsafe extern "C" fn(args: *const u64, rt: *mut RtContext) -> u64;
pub struct OpDecl { pub name: &'static str, pub params: Vec<BoundaryType>, pub ret: BoundaryType, pub deterministic: bool, pub func: OpFn,
    pub vectors: &'static [KnownAnswer] }   // (D40)
pub struct ProviderDecl { pub path: &'static str, pub crate_name: &'static str, pub crate_version: &'static str, pub ops: Vec<OpDecl>, pub thread_safe: bool, pub transport: Option<TransportDecl> }
pub struct ProviderRegistry { pub providers: Vec<ProviderDecl> }
#[repr(C)] pub struct RtContext { pub trap: u32, pub mode: u32, pub fuel: u64, pub depth: u64, pub max_depth: u64, pub mem_limit: u64, pub mem_live: u64, pub tracker: *mut AllocTracker,
                                  pub site: u64, pub chain: *mut u32 }  // field order is ABI; (D27) site at offset 56, 0 = none; (D28) chain at offset 64, null = no call chain
// cprime_provider::rt (D27): names and locations only, never program values (SA-61)
pub struct SiteEntry { pub spec: &'static str, pub file: &'static str, pub line: u32, pub column: u32, pub op: &'static str }   // op empty except for Perform sites
pub fn set_report_tables(sites: &'static [SiteEntry], functions: &'static [SiteEntry]) -> bool;   // false (ignored) when called a second time
// Runtime functions (C ABI): cprime_rt_alloc(rt, payload_bytes, len, tag, kind, ptr_mask) -> u64; cprime_rt_trap(rt, code); cprime_rt_trap_at(rt, code: u32, site: u32) (D27)
// Heap header: [0] rc u64 · [8] len u32 · [12] tag u16 · [14] kind u8 (0 bytes, 1 Vec, 2 struct, 3 variant) · [15] reserved · [16] ptr_mask u64 · [24] payload
// Release exit codes: 0 ok · 2 input invalid · 3 trap · 4 CPU-time limit (or limit not settable) · 5 memory limit · 6 provider panic · 7 call-depth limit
// cprime_rt_release_init(rt: *mut RtContext, cpu_time_ms: u64, memory_bytes: u64, max_call_depth: u64, workers: u64) -> u32   (0 ok, 1 failure)
```

Cross-crate functions called (§2E): `cprime_daemon::assurance::effective(db: &dyn Db, spec: DefId) -> Option<EffectiveEvidence>` (D39), `cprime_daemon::db::CprimeDatabase::open(root, config, providers: Vec<ProviderInfo>, registry: Option<ProviderRegistry>) -> Result<CprimeDatabase, OpenError>` (D9; `cprimec` always passes `None`), `CprimeDatabase::sync`, `cprime_daemon::pipeline::front_end(db, spec_name, source) -> FrontEnd`, `cprime_core::namespace::namespace_table`, `cprime_core::paths::impl_file`, `cprime_core::config::load_config`, `cprime_core::ir_verify::verify_function`, `cprime_types::check_spec`, `cprime_types::lower`, `cprime_effects::check_bindings(db) -> BindingCheck`, `cprime_memory::infer_borrows`, `cprime_memory::insert_rc`, `cprime_codegen::aot::emit_object(program: &Program, entry: DefId, out: &Path, opts: &AotOptions) -> Result<AotOutput, CodegenError>` (D28), `cprime_core::ast::node_index` (D26), `cprime_core::db::Db::impl_status`.

Configuration keys read (§2D): `project.name`, `providers[].crate|version|path|provider`, `limits.cpu_time_ms`, `limits.memory_bytes`, `runtime.cprime_source`, `log.level`, `release.call_chain` (bool, default `false`; D28, D29).

#### Public Interface

`cprimec` has no library API. Its interface is the command line (`clap` derive, `cprimec --help` generated from it).

| Command | Flags | Effect | stdout |
|---|---|---|---|
| `cprimec daemon` | `--project <dir>` (default `.`), `--config <k>=<v>` (repeatable), `--log-level <level>` | Launcher (SA-08): rebuild the runtime if the fingerprint is stale, then `execv` `.cprime/bin/cprimed --project <abs root> [--config …] [--log-level …]` | Nothing from `cprimec`; after `execv`, stdout belongs to `cprimed` |
| `cprimec runtime build` | `--project`, `--config`, `--force` | Generate `.cprime/runtime/`, build it, install `.cprime/bin/cprimed`, `.cprime/lib/libcprime_runtime.a`, `.cprime/providers.json`, write `.cprime/runtime.fingerprint`. Without `--force`, does nothing when the fingerprint is current | One line: `runtime up to date` or `runtime built: <fingerprint hex>` |
| `cprimec build --release` | `--project`, `--config`, `--out <dir>` (default `<root>/target/cprime/release`) | Release build (§Internal Logic D) | Two lines: binary path, inventory path |
| `cprimec inventory` | `--project`, `--format json\|text` (default `text`) | Provider inventory of the current runtime | Inventory (JSON object, or one line per provider) |
| `cprimec status` | `--project`, `--format json\|text` (default `text`) | Implementation status of every spec and entry | JSON array, or one line per definition |
| `cprimec --version` | — | Print `cprimec <crate version>` | that line |

`--release` is mandatory for `build` in v1; `cprimec build` without it is a usage error. All progress messages and all Cargo output go to stderr, never stdout.

**Exit codes of `cprimec`:**

| Code | Meaning |
|---|---|
| 0 | Success (`status`: every definition `current`) |
| 1 | Project check failure: `status` found a non-current definition; release refused (entry count, bindings, stale/absent/invalid implementation, T0 re-check diagnostics); provider set in the built runtime disagrees with `cprime.toml` |
| 2 | Usage error or `ConfigError` |
| 3 | Environment error: `.cprime/` not creatable, `.cprime/build.lock` held, `execv` failed |
| 4 | Toolchain error: `cargo` not found, `cargo build` or `cargo metadata` failed, LLVM object emission failed |
| 5 | Internal error: IR verifier failure or invariant breach (A-17) |

**Inventory JSON** (`<out>/<project.name>.inventory.json` and `cprimec inventory --format json`):

```json
{
  "format": "cprime-inventory-v1",
  "binary": "auth_service",
  "cprimec_version": "0.1.0",
  "target": "aarch64-apple-darwin",
  "providers": [
    {"provider": "host::Crypto", "crate": "cprime-host-crypto", "version": "0.1.0",
     "source": "registry+https://github.com/rust-lang/crates.io-index", "license": "MIT OR Apache-2.0"},
    {"provider": "acme::Zstd", "crate": "cprime-zstd-provider", "version": "0.3.1",
     "source": "path+/home/u/proj/providers/zstd", "license": null}
  ],
  "operations": [
    {"provider": "host::Crypto", "op": "sha256", "deterministic": true, "vectors": 2},
    {"provider": "host::Crypto", "op": "hmac_sha256", "deterministic": true, "vectors": 6},
    {"provider": "host::Crypto", "op": "blake3", "deterministic": true, "vectors": 2},
    {"provider": "acme::Zstd", "op": "compress", "deterministic": true, "vectors": 0}
  ],
  "evidence": {"root": "Main::main", "own": "proved", "effective": "tested",
               "tested": ["Auth::parse_header"], "assumed": ["Auth::CryptoCap::hmac_sha256"]}
}
```
`binary` is `null` for `cprimec inventory`. `operations` (D40) lists every operation of every provider in `flat_ops` order with its `deterministic` flag and known-answer vector count; providers are described as checked, never as verified (ADR-054). `evidence` (D39) is present only in a release inventory: `root` is the entry or the server's handler, and the other members are `assurance::effective(root)` without `unaccepted` (a release with an unaccepted reachable implementation is refused at step D.6); names and levels only, no program value (ADR-048). `cprimec inventory` writes no `evidence`. Readers of `cprime-inventory-v1` must ignore members they do not know; `operations` and `evidence` are additive members of v1. `target` is `x86_64-unknown-linux-gnu` or `aarch64-apple-darwin`, from `std::env::consts::{ARCH, OS}`. No timestamp is written (reproducible output).

**Status JSON** (`cprimec status --format json`): array sorted by qualified name, each `{"name": "Auth::generate_token", "kind": "spec"|"entry", "impl_file": "auth/generate_token.opq", "implementation": ImplStatus}`, plus `"effective": EffectiveEvidence` when the status is `current` (D39). Text form: `<name>  <status>[  <evidence>[ (effective <level>: <names>)]]  <impl_file>`, where the parenthesis appears only when `effective` differs from `own`, and `<names>` is `tested` followed by `unaccepted` names, joined by `, `.

##### Example

```text
$ cprimec build --release --project ~/proj
cprimec: runtime up to date (fingerprint 6c0e…41af)
cprimec: checking 3 implementations reachable from Main::main
cprimec: emitting object .cprime/release/entry.o
   Compiling auth_service v0.0.0 (/home/u/proj/.cprime/release)
    Finished `release` profile [optimized] target(s) in 3.84s
/home/u/proj/target/cprime/release/auth_service
/home/u/proj/target/cprime/release/auth_service.inventory.json
$ echo $?
0
$ echo '{"user_id": 7, "now": 100, "secret": "0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f"}' \
    | ~/proj/target/cprime/release/auth_service
{"user_id":7,"expires_at":3700,"sig":"0x5e0b…c2"}
```

Agent host configuration (MCP client) for a project:
```json
{"command": "cprimec", "args": ["daemon", "--project", "/home/u/proj"]}
```

#### Internal Logic

**A. Common start**

1. Parse arguments with `clap`. Errors → `clap`'s message on stderr, exit 2.
2. Resolve `--project` with `canonicalize`; it must contain `cprime.toml` (no upward search). `load_config(root, overrides)`; `Err` → stderr `cprimec: config: <e>`, exit 2.
3. Configure stderr logging at `--log-level` or `log.level` (Logging cross-cutting spec, target prefix `cprimec::`).
4. `create_dir_all(<root>/.cprime)`; failure → exit 3.

**B. Runtime fingerprint (`fingerprint::compute`)**

Blake3 over the concatenation of, in order:
1. ASCII `cprime.runtime.v1` and one zero byte.
2. `cprimec` version (`env!("CARGO_PKG_VERSION")`) and one zero byte.
3. `runtime.cprime_source` canonical absolute path, or the empty string, and one zero byte.
4. For each `[[providers]]` entry in file order: `crate`, zero byte, `version` or `path:` + canonical absolute path, zero byte, `provider`, zero byte.
5. For each path provider in file order: every regular file under `<path>/src/` plus `<path>/Cargo.toml`, sorted by relative path, contributing relative path, zero byte, 8-byte little-endian length, contents.
6. The bytes of `<root>/.cprime/runtime/Cargo.lock` if the file exists, else nothing.

The fingerprint is current when `<root>/.cprime/runtime.fingerprint` contains exactly the 64-character lower-case hex of this hash, and `.cprime/bin/cprimed`, `.cprime/lib/libcprime_runtime.a` and `.cprime/providers.json` exist. Changes inside `runtime.cprime_source` are not hashed; after editing C′ itself, run `cprimec runtime build --force`.

**C. `cprimec runtime build`**

1. Take an exclusive non-blocking `flock` on `<root>/.cprime/build.lock` (mode 0600). Held → stderr `cprimec: another runtime or release build is running`, exit 3. Released at exit.
2. Without `--force`, if the fingerprint is current, print `runtime up to date` and exit 0.
3. Validate `cprime.toml` providers: `provider` paths unique and not one of `host::Crypto`, `host::Clock`, `host::Random`, `host::Log` (else exit 2, message names the duplicate). `path` entries must exist and contain `Cargo.toml` (else exit 2).
4. Generate `<root>/.cprime/runtime/` (§E). Files are written with temporary name then rename; `Cargo.lock` from a previous build is kept so dependency versions do not drift.
5. Locate Cargo: the `CARGO` environment variable if set by an enclosing Cargo, else `cargo` on `PATH`. Missing → stderr `cprimec: cargo not found on PATH (A-11)`, exit 4.
6. Run `cargo build --release --manifest-path <root>/.cprime/runtime/Cargo.toml --target-dir <root>/.cprime/runtime/target --message-format human` with stdin closed, stdout and stderr both redirected to `cprimec`'s stderr (stdout of a launcher is the JSON-RPC channel). Non-zero exit → stderr `cprimec: runtime build failed (cargo exit <code>)`, exit 4; `.cprime/runtime.fingerprint` is left unchanged (A-15).
6a. Known-answer check (D40, ADR-054): run `<root>/.cprime/runtime/target/release/cprimed --check-providers` (stdin closed, stdout and stderr to `cprimec`'s stderr, 30 s timeout). Exit 1 → stderr `cprimec: runtime refused: a provider failed a known-answer vector` after `cprimed`'s own lines, exit 1; nothing is installed and `.cprime/runtime.fingerprint` is left unchanged. Timeout or any other exit → stderr `cprimec: provider check did not complete (<exit or timeout>)`, exit 4. For each `deterministic` operation of a non-built-in provider with no vectors (read after step 8 from `providers.json`), log `warn` `msg="deterministic operation has no known-answer vectors"` with `provider` and `op`.
7. Install: copy `target/release/cprimed` to `.cprime/bin/cprimed.tmp`, `chmod 0755`, rename to `.cprime/bin/cprimed`; copy `target/release/libcprime_runtime.a` the same way to `.cprime/lib/libcprime_runtime.a`.
8. Run `.cprime/bin/cprimed --describe-providers` (stdout captured, 10 s timeout). Non-zero exit → exit 1 with its stderr. Parse JSON; check that providers 4… match `cprime.toml` entries by crate name and provider path in order (else exit 1, `provider crate <c> declares path <p1> but cprime.toml says <p2>`). Write it to `.cprime/providers.json` (temp + rename).
9. Recompute the fingerprint (now including the new `Cargo.lock`) and write `.cprime/runtime.fingerprint` last (temp + rename). Print `runtime built: <hex>`; exit 0.

**D. `cprimec build --release`**

1. Lock `.cprime/build.lock` as in C.1. If the fingerprint is stale, run C.3–C.9 first (stderr `cprimec: runtime stale; rebuilding`).
2. Read `.cprime/providers.json` into `Vec<ProviderInfo>` (one per provider; `ProviderOpInfo.flat_index` from the JSON `index`; `ProviderInfo.thread_safe` from the JSON `thread_safe`; `ProviderInfo.transport` from the JSON `transport`, `null` → `None` (D30); boundary types parsed into `BoundaryTy`). No `ProviderRegistry` and no function pointer is built in `cprimec` (D9).
3. `CprimeDatabase::open(root, config, providers, None)`; `sync()`. Open/sync error → exit 3.
4. Root: collect `NsKind::Entry` and `NsKind::Server` entries of `namespace_table`. Count ≠ 1 → stderr `cprimec: release needs exactly one entry or server, found <n>` (names listed), exit 1. For a server, the reachable set of step 6 starts from the handler spec, and the transport provider must appear in `providers` (else exit 1, as step 9).
5. `check_bindings(db)`: any diagnostic → render each (§G) and exit 1.
6. Reachable set: worklist from the entry `DefId`; for each `d`, `impl_status(d)` must be `Current` — otherwise collect `<name>: <status>` and continue; then `accepted_impl(d)`'s function `Call` targets join the worklist. If any is non-current → stderr lists them, exit 1. Release builds never run T2/T3: they trust the acceptance header, whose context hash is current.
7. T0 re-check: for each reachable `d` in name order, read its `.opq` text from the database and run `pipeline::front_end(db, name, text)` with the project-relative `.opq` path as the file name. Any diagnostic → render, exit 1. Keep each reachable definition's parsed `ast::Impl` (`FrontEnd.imp`) and its `.opq` path for step 11 (trap site and function tables, D27).
8. IR: for each reachable `d`, take `accepted_impl(d).function` (pre-RC) and its layouts from `cprime_types::lower`. `borrows = infer_borrows(&functions, &BTreeMap::new())`. For each function `insert_rc` then `verify_function(.., RcInserted)`; error → exit 5.
9. Capability coverage: every `(cap, op)` in a `Perform` of a reachable function must map through `check_bindings` to a provider operation with that name → `OpTarget::Provider { flat_index }`. Missing binding → stderr `cprimec: capability <Cap> is performed but has no bind`, exit 1. `unimplemented` is empty by step 6.
10. Assemble `Program` (schema merged from `check_spec` of every reachable spec). `aot::emit_object(&program, entry, <root>/.cprime/release/entry.o, &AotOptions { call_chain: config.release.call_chain, trap_sites: true })` (D28; `trap_sites` is always `true` here); `Err` → exit 4 with the error text. The object defines `cprime_entry` with the C signature `extern "C" fn(rt: *mut RtContext, a0: u64, …) -> u64` and calls provider operations through the pointer in `cprime_rt_op_table`.
11. Generate `.cprime/release/` (§F) and run `cargo build --release --manifest-path <root>/.cprime/release/Cargo.toml --target-dir <root>/.cprime/release/target` with output redirected to stderr. Failure → exit 4.
12. `create_dir_all(--out)`; copy `target/release/<project.name>` to `<out>/.<name>.tmp`, `chmod 0755`, rename to `<out>/<project.name>`.
13. Inventory (§H) → `<out>/<project.name>.inventory.json` (temp + rename). Print the two paths; exit 0. No partial binary is ever left at `<out>/<project.name>` because only the final rename creates it (A-15). For a server, the inventory marks the transport provider with `"transport": true`.

**E. Generated runtime package — `<root>/.cprime/runtime/`**

`Cargo.toml`:
```toml
[package]
name = "cprime-runtime-<project.name>"
version = "0.0.0"
edition = "2024"
publish = false

[workspace]            # empty: the package is never captured by an enclosing workspace

[lib]
name = "cprime_runtime"
path = "src/lib.rs"
crate-type = ["staticlib", "rlib"]

[[bin]]
name = "cprimed"
path = "src/main.rs"

[dependencies]
# C′ crates: "=<cprimec version>" from crates.io, or { path = "<cprime_source>/crates/<name>" }
cprime-daemon = "=0.1.0"
cprime-provider = "=0.1.0"
cprime-host-crypto = "=0.1.0"      # path form: "<cprime_source>/providers/cprime-host-crypto"
cprime-host-clock = "=0.1.0"
cprime-host-random = "=0.1.0"
cprime-host-log = "=0.1.0"
# one line per [[providers]] entry, in file order:
cprime-zstd-provider = "=0.3.1"    # or { path = "<canonical absolute path>" }

[profile.release]
panic = "unwind"                 # provider panics are caught and become ProviderPanic
```

`src/lib.rs` (generated; `<ident>` is the crate name with `-` replaced by `_`):
```rust
use cprime_provider::{OpFn, ProviderRegistry};
use std::sync::atomic::{AtomicPtr, Ordering};

pub fn registry() -> ProviderRegistry {
    ProviderRegistry { providers: vec![
        cprime_host_crypto::cprime_provider(),
        cprime_host_clock::cprime_provider(),
        cprime_host_random::cprime_provider(),
        cprime_host_log::cprime_provider(),
        cprime_zstd_provider::cprime_provider(),   // one per [[providers]] entry
    ] }
}

/// Pointer to the flat operation table; AOT code loads it and calls entry k indirectly.
#[unsafe(no_mangle)]
pub static cprime_rt_op_table: AtomicPtr<OpFn> = AtomicPtr::new(std::ptr::null_mut());

/// Called once by the generated release main before cprime_entry.
pub fn init_op_table() {
    let reg = registry();
    let table: Vec<OpFn> = reg.flat_ops().iter().map(|(_, op)| op.func).collect();
    cprime_rt_op_table.store(Box::leak(table.into_boxed_slice()).as_mut_ptr(), Ordering::Relaxed);
}
```
Each provider crate exposes `pub fn cprime_provider() -> cprime_provider::ProviderDecl` at its crate root; this is the convention the generated code relies on.

`src/main.rs`:
```rust
fn main() -> std::process::ExitCode { cprime_daemon::run(cprime_runtime::registry()) }
```

**F. Generated release package — `<root>/.cprime/release/`**

`Cargo.toml`: package `name = "<project.name>"`, `version = "0.0.0"`, `edition = "2024"`, empty `[workspace]`, one `[[bin]]` named `<project.name>`, dependencies `cprime-runtime-<project.name> = { path = "../runtime" }`, `cprime-provider` (same form as §E), `serde_json = "=<version pinned in cprimec's own Cargo.lock>"`, `libc = "=<pinned>"`, and `build = "build.rs"`; `[profile.release] panic = "unwind"`.

`build.rs`: `println!("cargo:rustc-link-arg=<absolute path of entry.o>"); println!("cargo:rerun-if-changed=<same path>");`

`src/main.rs` for a **server** (D35, ADR-051): declares `cprime_handler` (`extern "C-unwind"`) and the `SITES`/`FUNCTIONS` tables as in 1a; `main` calls `cprime_runtime::init_op_table()`, `set_report_tables`, `cprime_rt_release_init(rt, 0, memory_bytes, max_call_depth, workers)` (no CPU limit), then `cprime_rt_serve(rt, cprime_handler as u64, <request field count>, <response field count>, <response pointer mask>, <transport index>, <transport config bytes>, <len>)`, and exits with its result. It reads no standard input. The transport configuration is the `[transport]` table of `cprime.toml`, passed as JSON bytes.

`src/main.rs` for a batch entry, generated from the entry's `TSpec` and `TypeSchema` (`release_main_gen.rs`):
1. Declares `unsafe extern "C" { fn cprime_entry(rt: *mut RtContext, a0: u64, …) -> u64; }` with one `u64` per entry parameter.
1a. Declares two `static` arrays of `cprime_provider::rt::SiteEntry` (D27): `SITES`, one entry per element of `AotOutput.sites` in that order, and `FUNCTIONS`, one entry per element of `AotOutput.functions` in that order. Each entry is filled as follows:
   - `spec`: the qualified spec name of the site's `func` (for `FUNCTIONS`, of the function);
   - `file`: the project-relative `.opq` path of that function's accepted implementation (kept in D.7);
   - `line`, `column`: from `node_index(imp).spans[node]` of that implementation's `ast::Impl` — the site's `node`; node 0 for `Entry` sites and for every `FUNCTIONS` entry;
   - `op`: `<capability qualified name>::<op>` for `Perform` sites (for example `Main::Boom::boom`), otherwise the empty string.

   String literals are written with Rust string escaping; the tables hold names and locations only, never a program value (SA-61).
2. `main`: calls `cprime_runtime::init_op_table()`, then `cprime_provider::rt::set_report_tables(&SITES, &FUNCTIONS)` (D27); allocates an `RtContext` on the heap (`Box::leak`) with `site: 0` and `chain: null_mut()`; when `release.call_chain` is `true`, it allocates `limits.max_call_depth + 1` `u32` entries, zero-filled (`Box::leak` of a boxed slice), and stores their address in `rt.chain` (D28); then it calls `cprime_rt_release_init(rt, <limits.cpu_time_ms>, <limits.memory_bytes>, <limits.max_call_depth>, <limits.workers>)` (D20), which stores the `par_map` worker count, sets `RtContext { trap: 0, mode: 0, fuel: u64::MAX, depth: 0, max_depth: <limits.max_call_depth>, mem_limit: <limits.memory_bytes>, mem_live: 0, tracker: null_mut() }`, installs the `SIGXCPU` action (exit 4) and sets `RLIMIT_CPU` to `ceil(limits.cpu_time_ms / 1000)` seconds. A return value of 1 → stderr `cprime: cannot set cpu time limit`, exit 4. The four limit values are written into the generated source as literals. Steps 3–5 run on one spawned thread (`std::thread::Builder::stack_size(<limits.max_call_depth> × 1024 + 1048576)`, D6); `main` joins it and exits with its code.
3. Reads all of stdin; `serde_json::from_slice`; not an object, missing key, extra key, or value not matching the parameter type → stderr `cprime: invalid input: <json path>: expected <type>`, exit 2.
4. One generated function per distinct type reachable from the parameter and result types: `decode_<n>(j: &serde_json::Value, rt) -> Result<u64, String>` and `encode_<n>(w: u64) -> serde_json::Value`. Scalars map to the word directly (`Bool` 0/1, `U8` zero-extended, `I64` two's complement, `Unit` 0). Heap types call `cprime_rt_alloc(rt, payload_bytes, len, tag, kind, ptr_mask)` and write the payload at offset 24 (bytes for `Bytes`/`FixedBytes`, words for `Vec`/struct/variant); encoding reads the header (`len` at 8, `tag` at 12) and payload. Hex strings follow the Value JSON encoding (SA-17): `"0x"` prefix, even length, `FixedBytes[N]` exactly N bytes.
5. Calls `cprime_entry`, encodes the result, writes it plus `\n` to stdout, flushes, exits 0. Traps, limits and provider panics exit inside the runtime (`rt.mode == 0`) with codes 3–7 (7 = call-depth limit). The stderr lines carry the spec and source location from `SITES`, and in `release.call_chain` builds the call chain from `FUNCTIONS`, exactly as `components/cprime-provider.md` step 8 specifies: for example `cprime: trap Overflow at auth/generate_token.opq:3:13 in Auth::generate_token` (exit 3), `cprime: call depth limit exceeded in Main::main (main/main.opq:2:7)` (exit 7), `cprime: provider panic in Main::Boom::boom at main/main.opq:3:3 in Main::main` (exit 6). Exit codes and the leading text of each line are unchanged (ADR-048).

**G. Diagnostics rendering (stderr)**

One line per diagnostic: `<file>:<line>:<column>: <kind> (<tier>): <message>` when `span` is present, else `<spec name>: node <n>: <kind> (<tier>): <message>`; `clause` appended as ` [clause: <text>]`.

**H. Inventory**

1. Run `cargo metadata --format-version 1 --manifest-path <root>/.cprime/runtime/Cargo.toml --locked` (stdout captured). Failure → exit 4.
1a. Operations (D40): for each provider in `.cprime/providers.json` order and each of its operations in `index` order, emit `{provider, op, deterministic, vectors}` into `operations`.
1b. Evidence (D39, release builds only): `assurance::effective(db, root)` with the database of step D.3; emit `evidence` = `{root: <qualified name>, own, effective, tested, assumed}`.
2. For each provider in `.cprime/providers.json` order, find the package whose `name` equals the provider's crate name and whose `id` is in the resolved dependency graph of the runtime package. Emit `provider` (provider path), `crate`, `version` (resolved, equal to `Cargo.lock`), `source` (`package.source`, or `path+<directory of manifest_path>` when `source` is null), `license` (`package.license` or `null`).
3. `cprimec inventory` requires a current fingerprint (else stderr `cprimec: runtime stale; run cprimec runtime build`, exit 1).

**I. `cprimec daemon` (launcher)**

1. Steps A.1–A.4. If the fingerprint is stale or the artefacts are missing, run C.1–C.9 (all output on stderr); any failure exits with C's code and the agent host sees the server end.
2. Build the argument vector `[".cprime/bin/cprimed" absolute, "--project", <abs root>, then each "--config", "<k>=<v>", then "--log-level", <level> if given]`; `libc::execv`. `execv` returns only on failure → stderr `cprimec: exec <path>: <OS error>`, exit 3. Standard input, output and error are inherited unchanged; `cprimec` writes nothing to stdout before `execv`.

**J. `cprimec status`**

1. Providers: `Vec<ProviderInfo>` read from `.cprime/providers.json` if present, else an empty `Vec` (status does not depend on providers). `CprimeDatabase::open(root, config, providers, None)`, `sync`.
2. For each `NsKind::Spec` or `NsKind::Entry` entry, sorted by name: `impl_status(def)`, `paths::impl_file(name)`, and for a `current` status `assurance::effective(db, def)` (D39). Print per `--format`. Exit 0 if every status is `current`, else 1; effective evidence never changes the exit code.

#### Data Model

No database. On-disk layout owned by `cprimec` (all under `<root>/.cprime/`, which projects add to their version-control ignore list):

| Path | Content | Written by |
|---|---|---|
| `build.lock` | empty; `flock` target | `runtime build`, `build --release` |
| `runtime/` | generated runtime package, `Cargo.lock`, `target/` | `runtime build` |
| `bin/cprimed` | project daemon executable, mode 0755 | `runtime build` |
| `lib/libcprime_runtime.a` | static runtime library | `runtime build` |
| `providers.json` | `cprimed --describe-providers` output | `runtime build` |
| `runtime.fingerprint` | 64 hex characters, no newline | `runtime build` (last step) |
| `release/` | generated release package, `entry.o`, `target/` | `build --release` |

Output: `<out>/<project.name>` (mode 0755) and `<out>/<project.name>.inventory.json`. Removing `.cprime/` entirely is always safe: the next launcher run rebuilds it. No destructive data change exists.

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Unknown command or flag; `build` without `--release` | exit | 2 | stderr `error: unexpected argument '--relase' found` (clap) |
| `cprime.toml` missing or invalid | exit | 2 | stderr `cprimec: config: cprime.toml not found in /home/u/proj` |
| Duplicate or reserved provider path in `cprime.toml` | exit | 2 | stderr `cprimec: config: provider path host::Log is reserved for a built-in provider` |
| `.cprime/` not creatable | exit | 3 | stderr `cprimec: cannot create /home/u/proj/.cprime: Permission denied` |
| Build lock held | exit | 3 | stderr `cprimec: another runtime or release build is running` |
| `execv` of `.cprime/bin/cprimed` fails | exit | 3 | stderr `cprimec: exec /home/u/proj/.cprime/bin/cprimed: No such file or directory` |
| `cargo` not on `PATH` | exit | 4 | stderr `cprimec: cargo not found on PATH (A-11)` |
| Runtime `cargo build` fails | exit | 4 | stderr `cprimec: runtime build failed (cargo exit 101)`; fingerprint unchanged |
| Built runtime's providers disagree with `cprime.toml` | exit | 1 | stderr `cprimec: provider crate cprime-zstd-provider declares path acme::Zip but cprime.toml says acme::Zstd` |
| Release: entry count ≠ 1 | exit | 1 | stderr `cprimec: release needs exactly one entry, found 0` |
| Release: invalid `bind` | exit | 1 | stderr `main.cprime:4:1: TypeMismatch (t0): provider host::Crypto operation hmac_sha256 is not deterministic but CryptoCap::hmac_sha256 is` |
| Release: reachable implementation not current | exit | 1 | stderr `cprimec: not current: Auth::generate_token: stale` |
| Release: T0 re-check diagnostic | exit | 1 | rendered diagnostics (§G) |
| Release: performed capability without `bind` | exit | 1 | stderr `cprimec: capability Auth::CryptoCap is performed but has no bind` |
| Release: IR verifier failure after RC insertion | exit | 5 | stderr `cprimec: internal error: IR verifier: block 2: value v7 used before definition` |
| AOT object emission fails | exit | 4 | stderr `cprimec: LLVM: <error text>` |
| Release `cargo build` fails | exit | 4 | stderr `cprimec: release build failed (cargo exit 101)`; no binary at `<out>` |
| `cargo metadata` fails | exit | 4 | stderr `cprimec: cargo metadata failed (exit 101)` |
| `status`: any definition not current | exit | 1 | stdout status lines; stderr `cprimec: 2 of 5 definitions are not current` |
| `runtime build`: a provider fails a known-answer vector (D40) | exit | 1 | stderr `cprimed: provider host::Crypto operation hmac_sha256 fails known-answer vector 2: result differs`, then `cprimec: runtime refused: a provider failed a known-answer vector`; nothing installed |
| `runtime build`: provider check times out or crashes (D40) | exit | 4 | stderr `cprimec: provider check did not complete (timeout after 30 s)` |
| `limits.workers` above 256 in `cprime.toml` or `--config` | exit 2 | `ConfigError::Invalid` | stderr `invalid value for limits.workers: must be 0 … 256` |
| `release.call_chain` not a boolean in `--config` (in `cprime.toml`: TOML type error) | exit 2 | `ConfigError::Invalid` / `ConfigError::Toml` | stderr `invalid value for release.call_chain: expected bool` |

#### Acceptance Criteria (Gherkin)

```gherkin
Feature: cprime-cli

  Background:
    Given a project at "/tmp/p" with cprime.toml name "demo" and no [[providers]]
    And cargo is on PATH

  Scenario: Happy path — launcher builds the runtime once and execs cprimed
    Given "/tmp/p/.cprime" does not exist
    When the agent host runs "cprimec daemon --project /tmp/p"
    Then "/tmp/p/.cprime/bin/cprimed" and "/tmp/p/.cprime/runtime.fingerprint" exist
    And the running program answers {"jsonrpc":"2.0","id":1,"method":"ping"} with {"jsonrpc":"2.0","id":1,"result":{}}
    And nothing other than JSON-RPC replies was written to stdout

  Scenario: Launcher skips the build when the fingerprint is current
    Given "cprimec runtime build --project /tmp/p" has succeeded
    When the agent host runs "cprimec daemon --project /tmp/p"
    Then cargo is not invoked

  Scenario: Adding a provider makes the fingerprint stale
    Given the runtime is built
    When a [[providers]] entry for a local path crate is appended to cprime.toml
    Then "cprimec runtime build --project /tmp/p" prints "runtime built: " followed by 64 hex characters

  Scenario: Editing a path provider's source makes the fingerprint stale
    Given the runtime is built with a path provider at "/tmp/p/providers/zstd"
    When "/tmp/p/providers/zstd/src/lib.rs" is modified
    Then "cprimec runtime build" rebuilds the runtime

  Scenario: Release build produces binary and inventory
    Given module Main declares "entry main(user_id, now U64, secret Bytes) Auth::SessionToken { uses Auth::CryptoCap }" and "bind Auth::CryptoCap = host::Crypto"
    And Main::main and Auth::generate_token are accepted and current
    When the user runs "cprimec build --release --project /tmp/p"
    Then the exit code is 0
    And "/tmp/p/target/cprime/release/demo" is executable
    And "/tmp/p/target/cprime/release/demo.inventory.json" lists provider "host::Crypto" with crate "cprime-host-crypto"

  Scenario: Release binary reads JSON and writes JSON
    Given the release binary from the previous scenario
    When it is run with stdin {"user_id":1,"now":100,"secret":"0x0001…1f"}
    Then stdout is one JSON object whose "expires_at" is 3700
    And the exit code is 0

  Scenario: Edge case — release binary given a wrong field type
    When the release binary is run with stdin {"user_id":"one","now":100,"secret":"0x00"}
    Then the exit code is 2
    And stderr starts with "cprime: invalid input: $.user_id"

  Scenario: Edge case — status reports a stale implementation
    Given Auth::generate_token was accepted and its ensures clause was then edited
    When the user runs "cprimec status --project /tmp/p --format json"
    Then the exit code is 1
    And the entry for "Auth::generate_token" has implementation.status "stale"

  Scenario: Error path — release refused with a stale implementation
    Given Auth::generate_token is stale
    When the user runs "cprimec build --release --project /tmp/p"
    Then the exit code is 1
    And stderr contains "not current: Auth::generate_token: stale"
    And no file "/tmp/p/target/cprime/release/demo" exists

  Scenario: Error path — no entry point
    Given no module declares an entry
    When the user runs "cprimec build --release --project /tmp/p"
    Then the exit code is 1 and stderr contains "release needs exactly one entry, found 0"

  Scenario: Error path — runtime build fails, fingerprint unchanged
    Given the runtime is built
    And a path provider whose source does not compile is added to cprime.toml
    When the user runs "cprimec runtime build --project /tmp/p"
    Then the exit code is 4
    And ".cprime/runtime.fingerprint" has the same content as before

  Scenario: Error path — cargo missing
    Given PATH contains no cargo and CARGO is unset
    When the user runs "cprimec runtime build --project /tmp/p"
    Then the exit code is 4 and stderr contains "cargo not found on PATH"

  Scenario: Error path — build without --release
    When the user runs "cprimec build --project /tmp/p"
    Then the exit code is 2

  Scenario: Release binary stops at the CPU-time limit
    Given cprime.toml sets limits.cpu_time_ms = 1000
    And the accepted entry Main::main has "uses Diverge" and recurses without end
    When the release binary is run with stdin {}
    Then it exits with code 4 within 3 seconds

  Scenario: Error path — worker count out of range
    Given cprime.toml sets limits.workers = 300
    When cprimec build --release runs
    Then the exit code is 2
    And stderr is "invalid value for limits.workers: must be 0 … 256"

  Scenario: Happy path — release overflow names spec and location
    Given auth.cprime holds the revision-1 contract of Auth::generate_token (no "now <= U64::MAX - 3600" clause)
    And a test fixture writes "auth/generate_token.opq" as a current acceptance header line followed by the concept §4.2 implementation starting with "(impl Auth::generate_token [uid now secret]" on line 2, the header computed by the harness with format_header and context_hash and no verification run
    And module Main declares "entry main(user_id, now U64, secret Bytes) Auth::SessionToken { uses Auth::CryptoCap }" whose accepted body calls Auth::generate_token with its three parameters
    And "bind Auth::CryptoCap = host::Crypto"
    When "cprimec build --release --project /tmp/p" runs
    And the release binary is run with stdin {"user_id":1,"now":18446744073709551615,"secret":"0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f"}
    Then the exit code is 3
    And stderr is exactly "cprime: trap Overflow at auth/generate_token.opq:3:13 in Auth::generate_token\n"

  Scenario: Error path — exit 7 names the spec
    Given cprime.toml sets limits.max_call_depth = 1000
    And the accepted entry Main::main in "main/main.opq" has "uses Diverge" and calls itself on every path, its "(impl Main::main" form on line 2
    When the release binary is run with stdin {}
    Then the exit code is 7
    And stderr is exactly "cprime: call depth limit exceeded in Main::main (main/main.opq:2:7)\n"

  Scenario: Error path — exit 6 names the operation
    Given a path provider "acme::Panicky" whose operation "boom(data Bytes) Bytes" calls panic!("boom")
    And module Main declares "capability Boom { boom(data Bytes) Bytes }", "bind Main::Boom = acme::Panicky" and "entry main(data Bytes) Bytes { uses Main::Boom }"
    And the accepted "main/main.opq" is "(impl Main::main [data]" on line 2 and "  (perform Boom::boom data))" on line 3
    When the release binary is run with stdin {"data":"0x01"}
    Then the exit code is 6
    And the last stderr line is "cprime: provider panic in Main::Boom::boom at main/main.opq:3:3 in Main::main"

  Scenario: Edge case — chain build reports exits 4 and 5 with the chain; default build does not
    Given cprime.toml sets limits.memory_bytes = 1048576
    And the accepted entry Main::main in "main/main.opq" calls Main::blow with 0x01, and Main::blow in "main/blow.opq" has "uses Diverge" and calls itself with (++ b b)
    When "cprimec build --release --project /tmp/p --config release.call_chain=true" runs and the binary is run with stdin {}
    Then the exit code is 5
    And the first stderr line is "cprime: memory limit exceeded"
    And the second stderr line starts with "cprime: call chain (outermost first, "
    And the third stderr line is "cprime:   Main::main (main/main.opq:2:7)"
    And every later stderr line is "cprime:   Main::blow (main/blow.opq:2:7)"
    When the CPU-time scenario's project is built with "--config release.call_chain=true" and its binary is run with stdin {}
    Then the exit code is 4
    And the first stderr line is "cprime: cpu time limit exceeded"
    And the second stderr line starts with "cprime: call chain (outermost first, "
    When both projects are built without the override and their binaries are run with stdin {}
    Then each binary's stderr is exactly one line: "cprime: memory limit exceeded" with exit code 5, and "cprime: cpu time limit exceeded" with exit code 4

  Scenario: Edge case — par_map elements 3 and 7 overflow on every run
    Given cprime.toml sets limits.workers = 4
    And a test fixture writes "main/bump.opq" as a current acceptance header followed by "(impl Main::bump [x] (+ x 1))" on line 2, for spec Main::bump(x U64) U64 with no requires and "ensures result == x + 1"
    And the accepted entry Main::main(xs Vec[U64]) Vec[U64] has the body (par_map Main::bump xs)
    When the release binary is run 20 times with stdin {"xs":[0,1,2,18446744073709551615,4,5,6,18446744073709551615,8,9]}
    Then every run exits with code 3
    And every run's stderr is exactly "cprime: trap Overflow at main/bump.opq:2:22 in Main::bump (par_map element 3)\n"

  Scenario: Edge case — no input value on stderr
    Given the release binary of "Happy path — release overflow names spec and location", built with "--config release.call_chain=true"
    When it is run with the same stdin
    Then the exit code is 3
    And stderr contains none of "18446744073709551615", "0x00010203" and "000102030405"
    And no stderr line contains a JSON value or a hexadecimal byte string

  Scenario: Happy path — server release build
    Given the "api.cprime" server example with an accepted Api::handle and the sample transport crate listed in cprime.toml
    When cprimec build --release runs
    Then it exits 0, the binary reads no standard input, and the inventory marks acme::HttpTransport with "transport": true

  Scenario: Release test — server longevity and limits (RFC 06 criteria 5–9)
    Given the built sample server with limits.workers 4, limits.request_fuel 100000000 and limits.shutdown_grace_ms 10000
    When the release test suite sends concurrent, trapping, 1,000,000 sequential and over-fuel requests, then SIGTERM
    Then each behaves as in cprime-provider step 16 and the process exits 0

  Scenario: Error path — both an entry and a server
    Given a project with entry Main::main and server Api
    When cprimec build --release runs
    Then stderr lists both names after "cprimec: release needs exactly one entry or server, found 2" and the exit code is 1

  Scenario: Happy path — status shows effective evidence (RFC 07 criterion 5)
    Given Lib::a accepted with evidence "proved" calling Lib::b accepted with evidence "tested"
    When the user runs "cprimec status --project /tmp/p --format json"
    Then the entry for "Lib::a" has implementation.status "current" and effective {"own": "proved", "effective": "tested", "tested": ["Lib::b"], "unaccepted": [], "assumed": []}
    And the text form prints "Lib::a  current  proved (effective tested: Lib::b)  lib/a.opq"
    When Lib::b is resubmitted and accepted with evidence "proved" and status runs again
    Then Lib::a's effective is "proved" and its text line has no parenthesis

  Scenario: Release test — the inventory reports effective evidence and vectors (RFC 07 criteria 5, 7)
    Given the project of the previous scenario with entry Main::main calling Lib::a
    When cprimec build --release runs
    Then the inventory's evidence is {"root": "Main::main", "own": "proved", "effective": "tested", "tested": ["Lib::b"], "assumed": []}
    And its operations list hmac_sha256 with vectors 6, sha256 with vectors 2 and blake3 with vectors 2
    And no inventory member contains a program value

  Scenario: Error path — runtime build refuses a provider that fails a vector (RFC 07 criterion 7)
    Given a path provider whose deterministic operation declares a vector it does not meet
    When cprimec runtime build runs
    Then the exit code is 1 and stderr names the provider, the operation and the vector index
    And .cprime/bin/cprimed is not replaced and .cprime/runtime.fingerprint is unchanged
```

#### Performance, Security, Observability

- **Performance targets:** Launcher overhead with a current fingerprint ≤ 50 ms p95 for a project with no path providers (hash of `cprime.toml` entries and `Cargo.lock`, three `stat` calls, `execv`). Fingerprinting a path provider costs one read of its `src/` tree (≤ 20 ms for ≤ 1 MiB of source). Runtime and release builds are dominated by Cargo; no target beyond "no work when up to date". Release T0 re-check ≤ 5 ms per reachable implementation.
- **Security:** The trap site and function tables generated into a release binary hold spec names, file paths, lines, columns and operation names only; release stderr never carries a program value (SA-61, ADR-048). `cprimec` opens no socket and performs no authentication (A-1, A-5). Cargo runs with the user's own credentials and network settings; registry access is Cargo's concern. Generated packages contain only identifiers derived from `cprime.toml` values validated by `load_config` (`project.name` pattern, provider paths `Ident(::Ident)*`); crate names are written as TOML keys with quoting, so no TOML injection is possible. The launcher passes only validated flags to `execv`. Release binaries run single-threaded with `RLIMIT_CPU` and the heap limit from configuration. Every linked provider is listed in the inventory with its exact version and licence (SA-15, OQ-23 support).
- **Observability:** Structured stderr logs (targets `cprimec::launcher`, `cprimec::runtime`, `cprimec::release`, `cprimec::inventory`, `cprimec::status`) with fields `ts`, `level`, `target`, `msg`, `root`, `fingerprint`, `duration_ms`, `exit_code`; Cargo's own human output is passed through to stderr unchanged. No metrics.

#### Gaps

None.
