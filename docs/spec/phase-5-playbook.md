# Phase 5 — Generation Playbook

> **Spec set:** cprime
> **Mode:** greenfield
> **derivedFromHld:** 0.12.0
> **HLD-impact status:** adjudicated — see `phase-1-analysis.md` banner.
> **Status:** Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> **Last updated:** 2025-12-20

## Contents

- [How to use this playbook](#how-to-use-this-playbook)
- [Step 0 — Scaffolding](#step-0--scaffolding)
- [Roadmap Phase 1 — Front end](#roadmap-phase-1--front-end)
- [Roadmap Phase 2 — IR and memory](#roadmap-phase-2--ir-and-memory)
- [Roadmap Phase 3 — JIT and daemon](#roadmap-phase-3--jit-and-daemon)
- [Roadmap Phase 4 — Verification](#roadmap-phase-4--verification)
- [Roadmap Phase 5 — Fuzzing and release](#roadmap-phase-5--fuzzing-and-release)
- [Integration and verification (every phase)](#integration-and-verification-every-phase)

## How to use this playbook

- Each step is one `codegen` invocation (`/apply` of one breakdown phase). A step names the spec sections it implements; nothing outside them is generated.
- Steps run in the order listed. The order follows the Phase 2B DAG and the HLD roadmap, which forbids JIT compilation before reference counting exists.
- A step is complete only when its **gate** passes. Gates use the commands in [Integration and verification](#integration-and-verification-every-phase).
- Steps that extend a crate already started (for example `cprime-core` in Steps 1 and 6) add modules; they do not rewrite earlier modules.
- No step may begin while any consumed component spec has a non-empty *Gaps* section or the Phase 6 audit has not passed.

## Step 0 — Scaffolding

| Item | Detail |
|---|---|
| Workspace | Root `Cargo.toml` with `[workspace]` members `crates/*` and `providers/*`; `resolver = "3"`; `[workspace.package] edition = "2024"`, `license` left for OQ-23. |
| Pins | `[workspace.dependencies]` with exact `=` versions for: `salsa`, `chumsky`, `blake3`, `z3`, `cranelift-codegen`, `cranelift-frontend`, `cranelift-module`, `cranelift-jit`, `inkwell` (feature `llvm18-0`), `arbitrary`, `tokio`, `serde` (feature `derive`), `serde_json`, `toml`, `clap` (feature `derive`), `libc`, `sha2`, `hmac`, `getrandom` (SA-02). `Cargo.lock` committed. |
| Toolchain | `rust-toolchain.toml`: `channel` = the stable release current on 2025-12-20 (≥ 1.85.0), components `rustfmt`, `clippy` (SA-01). |
| Crate skeletons | Empty `lib.rs` for every crate in Phase 2B; `cprime-cli` with `src/main.rs`. Each crate's `Cargo.toml` lists only the workspace dependencies its spec names. |
| CI matrix | Two jobs: `x86_64-unknown-linux-gnu`, `aarch64-apple-darwin` (SA-03). Each installs Z3 (major 4, minor ≥ 12) and LLVM 18 from the platform package manager and runs the gate commands. |
| Lint config | `clippy` with `-D warnings`; `rustfmt` default style; `#![forbid(unsafe_code)]` in every crate except `cprime-provider`, `cprime-codegen`, `cprime-verify` (JIT calls), `cprime-daemon` (lock, signals), `cprime-cli` (`execv`) and the four host provider crates. |
| Test corpus directory | `tests/corpus/` at the repository root: one sub-directory per task, each a small C′ project (`cprime.toml`, `.cprime`, accepted `.opq`). Seeded with the concept example (`auth.cprime`, `auth/generate_token.opq`). |

**Gate:** `cargo build --workspace` and `cargo fmt --all -- --check` pass on both CI targets.

## Roadmap Phase 1 — Front end

| Step | Component | Spec sections implemented | Gate |
|---|---|---|---|
| 1 | `cprime-core` (front-end modules) | `components/cprime-core.md`: ids, ast, tast (incl. `TerminationInfo`), diag, value, evidence, accept, config, prelude, db, namespace, resolve, context, paths, hash, program (types only) | Unit tests for every error-table row and Gherkin scenario of those modules |
| 2 | `cprime-syntax` | `components/cprime-syntax.md` (whole spec) | Concept example parses; grammar golden tests; recovery tests report every error |
| 3 | `cprime-types` (checking) | `components/cprime-types.md`: `check_spec`, `check_impl`, monomorphisation, completeness, `capability_ops`, `verification_deps` | Example type-checks; `NonExhaustiveMatch` on a `Result` match missing `Err` |
| 4 | `cprime-effects` | `components/cprime-effects.md` (whole spec) | Variant performing an undeclared capability yields `UnauthorizedEffect`; termination tests |

**Phase exit (HLD roadmap Phase 1):** the concept example parses, type-checks and passes effect checks; an undeclared capability yields `UnauthorizedEffect`; a `match` on `Result` missing `Err` yields `NonExhaustiveMatch`. Run as integration test `tests/phase1_exit.rs` in `cprime-effects` (dev-dependencies on `cprime-syntax`, `cprime-types`).

## Roadmap Phase 2 — IR and memory

| Step | Component | Spec sections implemented | Gate |
|---|---|---|---|
| 5 | `cprime-provider` (runtime ABI) | `components/cprime-provider.md`: heap layout, `RtContext`, `RtTrap`, every `cprime_rt_*` function, `AllocTracker` | ABI unit tests incl. drop of a 1,000,000-element chain without native stack overflow |
| 6 | `cprime-core` (IR) | `components/cprime-core.md`: ir, ir_verify (`IrStage::Lowered`, `IrStage::RcInserted`) | One test per verifier rule |
| 7 | `cprime-types` (lowering) | `components/cprime-types.md`: `lower` | Lowered example passes `ir_verify` at `Lowered` |
| 8 | `cprime-memory` | `components/cprime-memory.md` (whole spec) | RC-inserted IR passes `ir_verify` at `RcInserted`; borrow inference tests |
| 9 | `cprime-codegen` (interpreter) | `components/cprime-codegen.md`: semantics table, `interp`, `marshal` | Interpreter results on the corpus |

**Phase exit (HLD roadmap Phase 2):** the interpreter runs the example; the allocation-counting harness (`AllocTracker`, runtime protocol 7) reports zero live objects after every corpus run.

## Roadmap Phase 3 — JIT and daemon

| Step | Component | Spec sections implemented | Gate |
|---|---|---|---|
| 10 | `cprime-codegen` (JIT) | `components/cprime-codegen.md`: `jit` | JIT-vs-interpreter differential test on the corpus: equal results, equal trap kinds, zero live objects |
| 11 | `cprime-provider` (SDK) | `components/cprime-provider.md`: `BoundaryType`, `OpDecl`, `ProviderDecl`, `ProviderRegistry`, `Provider` trait, argument helpers, `guard`, `validate_registry` | SDK unit tests |
| 12 | Built-in providers | `components/cprime-host-providers.md` (whole spec) | Known-answer tests |
| 13 | `cprime-daemon` (`get_context`, `check_impl`, MCP, persistence, sync) | `components/cprime-daemon.md` except the `submit_impl` T2/T3 path | Protocol tests over an in-memory stdio pair; T0/T1 latency benchmark |
| 14 | `cprime-cli` (launcher, runtime build, status) | `components/cprime-cli.md`: `daemon`, `runtime build`, `status` | End-to-end: `cprimec runtime build` on the corpus example produces `.cprime/bin/cprimed`; launcher round trip |
| 15 | H1 measurement harness | `phase-4-cross-cutting.md` § Testing and measurement: H1 | Report written for the 50-task corpus |

**Phase exit (HLD roadmap Phase 3):** T0 ≤ 5 ms and T1 ≤ 20 ms p95 on implementations of ≤ 200 IR instructions with a warm database; JIT output matches the interpreter on the corpus; H1 measured and reported whether confirmed or not; a sample third-party provider wrapping an existing Rust crate works end to end (bind, check, submit, release build, provider inventory — the release-build leg completes in Phase 5).

## Roadmap Phase 4 — Verification

| Step | Component | Spec sections implemented | Gate |
|---|---|---|---|
| 16 | `cprime-verify` (SMT) | `components/cprime-verify.md`: SMT encoding, obligations, scheduling, counterexamples, acceptance table rows without T3 | One test per acceptance-table row reachable without fuzzing |
| 17 | `cprime-daemon` (`submit_impl`) | `components/cprime-daemon.md`: `submit_impl`, acceptance header write, atomic persistence | Crash-injection test: no partial `.opq` |

**Phase exit (HLD roadmap Phase 4):** the concept §6.3 rejection is reproduced; the corrected implementation reaches `proved`; the revision-1 contract (no `now` bound) yields `ReachableTrap`.

## Roadmap Phase 5 — Fuzzing and release

| Step | Component | Spec sections implemented | Gate |
|---|---|---|---|
| 18 | `cprime-verify` (fuzzing) | `components/cprime-verify.md`: fuzzing harness, determinism probe, remaining acceptance-table rows | One test per T3 row incl. `fuzz_budget_incomplete` |
| 19 | `cprime-codegen` (AOT) | `components/cprime-codegen.md`: `aot` | AOT-vs-interpreter differential test on the corpus: equal results, equal trap kinds, equal trap locations (D27) |
| 20 | `cprime-cli` (release, inventory) | `components/cprime-cli.md`: `build --release`, `inventory`, generated release package | Release binary of the corpus example runs; inventory lists every linked provider with its `Cargo.lock` version |
| 21 | Benchmarks | `phase-4-cross-cutting.md` § Testing and measurement: benchmarks, trap-reporting overhead, witness-trace effectiveness | Reports written: release run time versus hand-written Rust, whether within 2× or not; overhead of default and `release.call_chain` builds; witness-trace effectiveness on the H1 corpus, whether traces help or not |

**Phase exit (HLD roadmap Phase 5):** performance target measured and reported; AOT output and trap locations match the interpreter on the corpus; trap-reporting overhead and witness-trace effectiveness reported.

### `par_map` parts per step (RFC 02)

Existing steps gain the `par_map` parts below; each step's exit criterion gains that part's scenarios from the named spec. No new step and no reordering.

| Steps | `par_map` part |
|---|---|
| 2, 3, 4 | front end: grammar and E-PARMAP; typing (D15); effect restrictions (D16) |
| 5 | runtime ABI: `cprime_rt_par_map` sequential path |
| 6 | IR rule `par-map` |
| 7 | lowering to `Inst::ParMap` |
| 8 | memory: `xs` borrowed, result owned (D18) |
| 9, 10 | interpreter and JIT |
| 11, 12 | `thread_safe` on `ProviderDecl`; built-in flags |
| 16 | SMT encoding (D17) |
| 19 | AOT, plus the parallel path of `cprime_rt_par_map` (worker contexts, copy, park-on-trap, shared memory counter) |
| 20 | `limits.workers` and the fifth `cprime_rt_release_init` argument |
| 21 | `par_map` benchmarks and speed-up report |

### RFC 03 parts per step

Existing steps gain the witness-trace (ADR-047) and release-trap-location (ADR-048) parts below; each step's exit criterion gains that part's scenarios from the named spec. No new step and no reordering.

| Steps | RFC 03 part |
|---|---|
| 1 | D22 witness types in `cprime_core::diag`; `ast::node_index` (D26); `verify.witness_max_steps` and `release.call_chain` (D29) in `load_config` |
| 5 | `RtContext.site` and `RtContext.chain`; `cprime_rt_trap_at`; `rt::set_report_tables` and `SiteEntry`; mode-0 location and call-chain lines (`cprime-provider` step 8) |
| 7 | `node_map` per D23 |
| 8 | preservation of D23 `node_map` entries through RC insertion |
| 9 | `interp::replay` with `ReplayOracle` (D24) |
| 16 | T2 witness construction (`cprime-verify` step 9a) |
| 17 | step spans and names from `node_index`; `timings_us.witness` in `submit_impl` |
| 18 | T3 witness construction (`cprime-verify` step 12a); stub log |
| 19 | AOT trap sites, `Entry` and `Perform` sites, inline `slice` check, call-chain prologue (`AotOptions`); differential trap-location check |
| 20 | site and function tables in the generated release main; `release.call_chain` chain buffer; worker chain buffers |
| 21 | overhead report for default and `release.call_chain` builds; witness-trace effectiveness report (`bench/witness/`) |

### RFC 06 parts per step (servers, ADR-051)

Existing steps gain the server parts below; each step's exit criterion gains that part's scenarios from the named spec. No new step and no reordering.

| Steps | RFC 06 part |
|---|---|
| 1 | `Limits.request_fuel`, `Limits.shutdown_grace_ms` in `load_config`; `Item::Server`, `NsKind::Server`, canonical tag `0x51` (D31, D36) |
| 2 | `server` item grammar and E-SERVER-LINE (`cprime-syntax`) |
| 4 | `check_bindings` B-11 and `InvalidServer` (`cprime-effects`) |
| 11 | `TransportDecl`, `RecvStatus`, `ProviderDecl.transport`, `describe_json` transport field (D30) |
| 12 | Built-in providers thread-safe (SA-69) |
| 19 | AOT server handler: unwind tables, `cprime_handler`, `C-unwind` trap entries (D34) |
| 20 | Server release package and inventory (D35); `cprime_rt_serve` with task heaps, mode 3, shutdown and exit codes 8/9 (D33); the sample HTTP transport crate |
| 21 | Benchmarks: programs that are not servers within 2% of their run time without the feature (G13); RFC 06 criteria 5–9 as release tests |

**Phase 5 exit gains:** the sample server passes RFC 06 criteria 5–9, and the single-threaded cost stays within 2%.

### RFC 07 parts per step (soundness boundaries, ADR-052 to ADR-055)

Existing steps gain the parts below; each step's exit criterion gains that part's scenarios from the named spec. No new step and no reordering.

| Steps | RFC 07 part |
|---|---|
| 1 | `Severity::Note`, `VacuousContract`, `VacuityUnknown`, `ConcreteCheck`, `ConcreteStatus`, `VacuityOutcome`, `EffectiveEvidence`, `UnacceptedCallee` in `cprime_core` (D37, D39, D43); `verify.provider_wait_ms` in `load_config` (D41); `ProviderOpInfo.vectors` (D40) |
| 8 | `RcOptions` and `insert_rc_opts` (`cprime-memory`, D45) |
| 9 | pre-RC execution in `interp::run`; Part 6 item 29 differential test (`cprime-codegen`, D45; RFC 07 criterion 16) |
| 10 | `JitModule` is `Send` (D41) |
| 11 | `OpDecl.vectors`, `KnownAnswer`, `Lit`, `sdk::check_vectors`, `describe_json` `vectors` (`cprime-provider` step 17, D40) |
| 12 | built-in known-answer vectors (`cprime-host-providers`; RFC 07 criterion 7) |
| 13 | `cprimed --check-providers`; `assurance::effective` (D39, D40) |
| 14 | runtime build step 6a; `cprimec status` effective evidence (D39, D40; RFC 07 criteria 5, 7) |
| 16 | vacuity check (step 6a), concrete replay (step 9b), `unsupported_construct` fail-closed reason (D37, D43, D44; RFC 07 criteria 1–3, 12–14) |
| 17 | vacuity cache, `effective` reply member, `-32001` `unsupported_construct` (D38, D39, D44; RFC 07 criteria 4, 6) |
| 18 | T3 provider thread, `fuzz_live`, evaluator `RtContext`, `provider_failure` and `provider_timeout`, T3 concreteness (D41, D42, D43; RFC 07 criteria 8–10) |
| 20 | inventory `operations` and `evidence` (D39, D40) |

**Phase 4 exit gains:** RFC 07 criteria 1–4 and 12–14 pass. **Phase 5 exit gains:** RFC 07 criteria 5–10 and 16 pass.

## Integration and verification (every phase)

Run in this order on both CI targets; a failure is a stop (Execute mode), not a retry with relaxed flags.

| Order | Check | Command | Pass criterion |
|---|---|---|---|
| 1 | Format | `cargo fmt --all -- --check` | exit 0 |
| 2 | Lint | `cargo clippy --workspace --all-targets -- -D warnings` | exit 0 |
| 3 | Build | `cargo build --workspace --all-targets` | exit 0 |
| 4 | Unit tests | `cargo test --workspace --lib` | exit 0 |
| 5 | Integration tests | `cargo test --workspace --test '*'` | exit 0 |
| 6 | Coverage | the active Rust Practice Pack's coverage command | ≥ 70% line coverage per crate (codegen rule C3) |
| 7 | Smoke test (from Step 14) | `cprimec status --project tests/corpus/auth` | exit 0, every spec `current` |
| 8 | Differential tests (from Step 9) | `cargo test -p cprime-codegen --test differential` | exit 0; from Step 9 the pre-RC and reuse legs (Part 6 item 29); from Step 10 the JIT leg; from Step 19 the AOT leg also compares trap locations |
| 9 | Leak harness (from Step 9) | `cargo test -p cprime-codegen --test leaks` | exit 0, zero live objects |
| 10 | Observability check (from Step 13) | `cargo test -p cprime-daemon --test logging` | every log line is one JSON object with the fields of `phase-4-cross-cutting.md` § Logging; no submitted source at `info` |
| 11 | Security scan | `secscan` skill over the workspace | no finding ranked high or critical |
