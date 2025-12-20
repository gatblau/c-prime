# C′ — Architectural Decisions

> **Mode:** draft
> **Generated from:** inline context — `docs/cprime-concept.md` (Draft, revision 2)
> **Revision:** 0.12.0
> **Last updated:** 2025-12-20

Append-only. ADR-012 to ADR-021 carry forward the ten concept decisions OQ-1 to OQ-10, all recorded as decided on 2025-12-20; the original ids are kept in brackets so the concept's cross-references still resolve. Alternatives marked *(drafted)* were not stated in the concept and are recorded for the user to confirm or correct.

### ADR-001: Two surfaces — humans write `.cprime` contracts, agents write `.opq` implementations
- **Decision.** Split every program into a declarative human surface (`.cprime`) and a compact S-expression implementation surface (`.opq`) that resolve to the same semantic graph.
- **Context.** The human gate moves from reviewing implementations to writing contracts; the implementation surface is optimised for agents (no imports, no type restatement, no indentation rules).
- **Consequences.** `.opq` cannot disagree with its spec about types or effects because it does not restate them. Humans are not expected to read `.opq`, so contract completeness carries the whole safety argument.
- **Alternatives considered.** One surface for both humans and agents *(drafted)* — rejected because agents would restate types and effects, adding tokens and allowing the two to drift.

### ADR-002: Memory safety from immutable values and Perceus RC, no borrow checker
- **Decision.** All values are immutable; `cprime-memory` inserts reference-counting operations at compile time and reuses memory in place (FBIP) when a count is 1.
- **Context.** Agents should never annotate lifetimes or ownership.
- **Consequences.** No cycles (a value can reference only values that existed before it), so RC reclaims all memory. Non-atomic counts tie v1 to single-threaded programs. Performance is a measured risk (2× Rust target).
- **Alternatives considered.** A borrow checker — rejected: agents would have to annotate ownership. Region inference — deferred to after Phase 5 benchmarks (non-goal).

### ADR-003: The daemon is a cache; files on disk are authoritative
- **Decision.** `cprimed` loads `.cprime`/`.opq` files on start-up; `check_impl` never writes; an accepted implementation is written atomically (temporary file, then rename) before the reply; rejected submissions are not persisted.
- **Context.** Agents work against in-memory state to keep the loop fast, but accepted code must survive restarts and be visible to version control.
- **Consequences.** Every acceptance is one file write and one self-contained diff. Recovery from any daemon failure is a restart.
- **Alternatives considered.** Daemon-owned store as source of truth *(drafted)* — rejected: hides accepted code from version control and makes the daemon stateful.

### ADR-004: Content-addressed definitions with a namespace table
- **Decision.** Each definition's `DefId` is the Blake3 hash of its canonical serialisation, with references replaced by `DefId`s and locals by de Bruijn indices; a namespace table maps qualified names to `DefId`s.
- **Context.** Incremental computation and identity must be stable under renaming of local bindings; RPC callers use names.
- **Consequences.** Renaming a local does not change a hash. A change to canonical serialisation changes every hash (OQ-14).
- **Alternatives considered.** Name-based identity *(drafted)* — rejected: renames and edits would invalidate more of the incremental cache.

### ADR-005: One effect model — a capability is a named set of effect operations
- **Decision.** Effect sets are sets of capabilities; no ambient authority; `Diverge` is a built-in capability; `.opq` cannot install handlers or call foreign code; providers are bound only in `.cprime`.
- **Context.** Revision 1 mixed effects and capability tokens and treated FFI as an effect.
- **Consequences.** The effect check is a subset test on transitive effect sets. No capability tokens at run time in v1.
- **Alternatives considered.** Capability tokens as runtime values — rejected for v1 (two models instead of one). FFI as an effect — rejected: foreign code would bypass the effect checker.

### ADR-006: Checked arithmetic and indexing; the verifier must prove traps unreachable
- **Decision.** `+ - *` trap on overflow, `/ %` on division by zero, `index` on out-of-bounds; explicit `wrapping_add`, `wrapping_sub`, `wrapping_mul` for modular arithmetic; every trap site is a proof obligation.
- **Context.** Revision 1's example overflowed `U64` silently.
- **Consequences.** Contracts must bound inputs (e.g. `now <= U64::MAX - 3600`); unbounded contracts yield `ReachableTrap`.
- **Alternatives considered.** Wrapping by default — rejected: silent overflow is the defect that prompted this decision.

### ADR-007: Evidence levels and a per-spec assurance policy
- **Decision.** Results are classified `proved` / `tested` / `none`; each spec declares `assurance proved` (default) or `assurance tested`; under `proved`, `unknown` is a rejection (`VerificationInconclusive`).
- **Context.** Revision 1 presented fuzzing as verification.
- **Consequences.** Acceptance is only as strong as the declared policy; fuzzing is a declared, weaker level.
- **Alternatives considered.** Treat fuzzing as equivalent to proof — rejected (the revision-1 defect). Proof only, no fuzzing — rejected: some specs would never be accepted.

### ADR-008: Cranelift JIT for iteration, LLVM AOT for release
- **Decision.** `cprime-codegen` uses Cranelift for `check_impl`/`submit_impl` and fuzzing, LLVM via `inkwell` for release builds.
- **Context.** T1 must stay ≤ 20 ms; release code must approach hand-written Rust.
- **Consequences.** Two back ends must agree; JIT-vs-interpreter differential tests guard the JIT. LLVM version churn is a risk.
- **Alternatives considered.** LLVM for both *(drafted)* — rejected: compile latency would not fit T1. Cranelift for both *(drafted)* — rejected: release performance target.

### ADR-009: LLM traffic is JSON text
- **Decision.** No binary encoding on the RPC/MCP interface.
- **Context.** Binary encodings shrink bytes but not the tokens a model reads.
- **Consequences.** `serde_json` only.
- **Alternatives considered.** MessagePack — rejected: no token saving.

### ADR-010: Rejections are results; the JSON-RPC error object is for protocol faults
- **Decision.** Compile and verification failures return `result` with `status: "rejected"`; `error` is reserved for unknown method, malformed parameters and daemon failure.
- **Context.** A rejection is a normal outcome of the agent loop and carries structured diagnostics.
- **Consequences.** Clients branch on `status`, not on transport-level errors.
- **Alternatives considered.** JSON-RPC `error` for compile errors *(drafted)* — rejected: conflates faulty requests with faulty code.

### ADR-011: C′-IR is block-argument SSA with explicit traps
- **Decision.** SSA with block arguments in place of phi nodes, terminators (`Return`, `Jump`, `Branch`, `Switch`, `Trap`), struct and variant construction, field access, checked arithmetic, RC instructions inserted only by `cprime-memory`; the content hash is computed over the canonical AST, not the IR.
- **Context.** Revision 1's IR could not express its own example; Cranelift uses the block-argument model.
- **Consequences.** Lowering to Cranelift is direct; `Switch` is exhaustive by construction; the IR has no type parameters.
- **Alternatives considered.** Phi-node SSA *(drafted)* — rejected: mismatches Cranelift's model.

### ADR-012 (OQ-1): Parse with `chumsky`
- **Decision.** `chumsky` for both surfaces.
- **Context.** One submission should report every parse error.
- **Consequences.** Error recovery is built in; parse time is a small share of T0.
- **Alternatives considered.** `winnow` — rejected: no built-in error recovery.

### ADR-013 (OQ-2): Fuzzing is a custom generator driving the JIT
- **Decision.** Custom generator inside `cprimed` using `arbitrary`, rejection sampling on `requires`, calling JIT code in memory.
- **Context.** Inputs must satisfy `requires`, effects must be stubbed, execution must be in memory.
- **Consequences.** Rejection sampling may fail for tight `requires` (OQ-21).
- **Alternatives considered.** An embedded off-the-shelf fuzzer — rejected: runs a separate binary over raw bytes.

### ADR-014 (OQ-3): One `.opq` file per implementation beside its module
- **Decision.** `auth.cprime` → `auth/generate_token.opq`.
- **Context.** Acceptance must be one atomic write and one self-contained diff.
- **Consequences.** File path is derivable from the qualified spec name.
- **Alternatives considered.** One `.opq` per module *(drafted)* — rejected: acceptance would rewrite shared files.

### ADR-015 (OQ-4): General user-declared sum types; `Result[T, E]` in the prelude
- **Decision.** User-declared sum types with exhaustive `match`; `Result` is an ordinary prelude sum type; generics are monomorphised before IR lowering.
- **Context.** Programs need sum types beyond failure.
- **Consequences.** The IR gains `MakeVariant`, `GetVariantField` and `Switch`; the grammar gains sum declarations.
- **Alternatives considered.** A built-in special-cased `Result` only — rejected: too narrow.

### ADR-016 (OQ-6): Runtime limits live in configuration, not `.cprime`
- **Decision.** CPU-time and memory limits are set in `cprimed`/`cprimec` configuration; exceeding either aborts the program.
- **Context.** Termination is already proved; limits guard only against slow or memory-heavy programs.
- **Consequences.** No grammar or verifier work for resource bounds in v1.
- **Alternatives considered.** Resource bounds as a `.cprime` language feature — rejected: cost exceeds v1 benefit.

### ADR-017 (OQ-7): `UnconstrainedOutput` blocks acceptance unless waived per field
- **Decision.** Every struct result field must be mentioned as `result.<field>` by an `ensures` clause or waived in `unconstrained [...]`.
- **Context.** The human must accept a contract gap explicitly.
- **Consequences.** Severity `error`; grammar gains the waiver clause; sum-typed result rules are fixed in `/spec` (OQ-19).
- **Alternatives considered.** Warning only — rejected: gaps would pass silently.

### ADR-018 (OQ-8): MCP tool names mirror the JSON-RPC methods
- **Decision.** `cprime_get_context`, `cprime_check_impl`, `cprime_submit_impl`, with arguments equal to the JSON-RPC `params`.
- **Context.** One vocabulary across both interfaces.
- **Consequences.** Less documentation and test surface.
- **Alternatives considered.** Separate MCP-specific argument schemas *(drafted)* — rejected: doubles the surface.

### ADR-019 (OQ-9): H1 measured on a 50-task corpus, same model and prompt for both languages
- **Decision.** 50 small, well-known algorithmic and systems tasks; tokens counted with the model's own tokeniser; first-submission acceptance rate compared.
- **Context.** Isolate the effect of the language from the effect of the model.
- **Consequences.** Phase 3 reports H1 whether confirmed or refuted.
- **Alternatives considered.** Different models or prompts per language *(drafted)* — rejected: confounds the comparison.

### ADR-020 (OQ-5, amended by OQ-10): Four built-in providers, no network or filesystem
- **Decision.** Ship `host::Crypto`, `host::Clock`, `host::Random`, `host::Log`; no dynamic loading; the catalogue is open through the SDK (ADR-021).
- **Context.** Keep the trusted computing base small while covering the examples, the H1 corpus and the benchmarks.
- **Consequences.** Network and filesystem access exist only if a project supplies its own provider.
- **Alternatives considered.** Built-in network and filesystem providers — rejected: enlarges the TCB.

### ADR-021 (OQ-10): Reuse Rust crates through a provider SDK, statically linked
- **Decision.** Providers are Rust crates on `cprime-provider`, one capability each, statically linked into a project-specific runtime built by `cprimec` with Cargo and used by both `cprimed` and release builds; boundary types are plain data; `bind` must match signatures and `deterministic` flags exactly; each release reports a provider inventory; fuzzing probes determinism.
- **Context.** `.opq` has no FFI; I/O and audited specialist code (TLS, compression, database drivers) are not worth rewriting in C′.
- **Consequences.** Every provider joins the TCB. Development and release run identical provider code. A rebuild is needed when the provider list or a provider crate changes.
- **Alternatives considered.** Direct FFI from `.opq` — rejected: bypasses the effect checker. Dynamic loading — rejected: non-goal.

### ADR-022: Perceus calling convention — inferred borrowing, owned by default
- **Decision.** `cprime-memory` infers, per function parameter, whether the callee borrows the value or takes ownership of it, following the borrowing extension of Perceus. Any parameter the inference does not mark as borrowed is owned. Return values are always owned. Neither `.cprime` nor `.opq` carries ownership annotations.
- **Context.** External review (2025-12-20, finding b): the concept specifies dup-before-use and drop-after-last-use within a block, but not ownership at calls and returns, which is Perceus's main design axis. Agents must never annotate ownership (ADR-002).
- **Consequences.** The convention is part of the code-generation ABI and must be identical in the Cranelift JIT and the LLVM AOT back end (ADR-008). Trap edges perform no cleanup, because a trap aborts the program. The provider boundary is unaffected, because boundary values are copied (ADR-021). Before Phase 2, `/spec` fixes RC handling on block arguments, `Switch` arms that bind variant payloads, the type of the `DropReuse` token, and where the count-is-1 test runs.
- **Alternatives considered.** Every parameter owned — simplest, but emits more `RcDup`/`RcDrop` pairs; kept as the fallback. Programmer-written borrow annotations — rejected: contradicts ADR-002.

### ADR-023: Acyclicity is a language invariant
- **Decision.** The language has no recursive value bindings (no `letrec` over values) and no construct by which a value can reference itself or a value constructed after it. Recursive type declarations are permitted; their values are built only from values that already exist. The grammar and `cprime-types` enforce this.
- **Context.** External review (2025-12-20, finding c): the acyclicity argument in concept §5.3 is the only justification for omitting a cycle collector, so it must be a stated rule, not an observation.
- **Consequences.** Reference counting reclaims all memory. Any future feature that could tie a knot — laziness, mutable references, self-capturing closures (see OQ-30) — must either preserve the invariant or add cycle collection. `cprime-types` also rejects a type that has no finite values (for example `type T = { t: T }`).
- **Alternatives considered.** Add a cycle collector — rejected for v1: unnecessary while the invariant holds, and it costs run time. Leave the property implicit — rejected: an implementation could break it without noticing.

### ADR-024 (amended by ADR-054): Termination rules
- **Decision.** There is no loop construct; iteration is recursion. Every recursive function whose effect set lacks `Diverge` carries a `decreases` clause. Mutually recursive functions — one strongly connected component of the call graph — share one lexicographic measure, a tuple of `U64` expressions, which must strictly decrease on every call inside the component. Measures are always written explicitly in v1. `Diverge` propagates through effect sets, including the effect sets carried by function types.
- **Context.** External review (2025-12-20, finding d): the concept states the single-function rule only, and the termination check is Phase 1 scope (`cprime-effects`), so the rules are needed before Phase 1.
- **Consequences.** The `.opq` grammar needs a `decreases` form that accepts a tuple. `MissingDecreases` and `NonDecreasing` apply per component. Obligation granularity (per clause or per call site) is fixed in `/spec`.
- **Alternatives considered.** Automatic measure inference, as Dafny does — deferred: adds heuristics, and agents can write measures. Single-function measures only — rejected: mutual recursion would need `Diverge`. Loops with invariants — rejected for v1: adds loop invariants to the verifier.
- **Amended.** The termination guarantee is relative: a call to a spec without `Diverge` cannot diverge, provided every provider call returns. A call ends in exactly one outcome — returns, traps, exceeds a resource limit (CPU time or fuel, memory, call depth), provider fails, or diverges (only with `Diverge`) — and each outcome maps to one release exit code (`08-interfaces.md`).

### ADR-025 (amended by ADR-052): Acceptance rules for a submission
- **Decision.** (1) Any obligation with a counterexample, from Z3 or from fuzzing, means rejection with evidence `none`. (2) Every obligation `unsat` means acceptance with evidence `proved`. (3) Otherwise at least one obligation is `unknown` — including a per-obligation timeout or a cut-off when the per-submission budget is spent — and: under `assurance proved` the submission is rejected with `VerificationInconclusive`; under `assurance tested` fuzzing runs, and the submission is accepted with evidence `tested` only if fuzzing completes its configured budget without a violation. Obligations of one submission run in parallel solver instances inside `cprimed`; when the per-submission budget is spent, the remaining obligations are cut off in a fixed order. The full policy × solver result × fuzz result table is fixed in `/spec`.
- **Context.** External review (2025-12-20, findings e and f): the concept defines the evidence levels but not every transition, and the T2 targets (≤ 250 ms per obligation, ≤ 2 s per submission) do not compose when obligations run one after another and number more than eight.
- **Consequences.** Budget exhaustion never yields acceptance under `assurance proved`; under `assurance tested` it leads to fuzzing. `cprimed` is concurrent internally even though compiled programs are single-threaded. OQ-21 still governs the case where fuzzing finds no inputs that satisfy `requires`.
- **Alternatives considered.** Treat budget exhaustion as a hard rejection under every policy *(drafted)* — rejected: `assurance tested` would become unusable for specs with many obligations. Run obligations sequentially *(drafted)* — rejected: the two T2 targets conflict.
- **Amended.** A vacuous or inconsistent contract rejects with `VacuousContract` before any obligation (ADR-052). Diagnostics of severity `note` never affect acceptance.

### ADR-026: Completeness rule for sum-typed results
- **Decision.** When a spec's result type is a sum type, at least one `ensures` clause must test which variant `result` is, and, for each variant, every field must be mentioned by an `ensures` clause or waived in `unconstrained`. A spec may waive the variant test itself only explicitly. Violations yield the same rejection as `UnconstrainedOutput`. The syntax for variant tests and variant-field waivers is fixed in `/spec`.
- **Context.** External review (2025-12-20, §3): with sum-typed results deferred, an implementation returning `Err` for every input satisfies any `ensures` clause guarded by the `Ok` case — the obvious way to game acceptance. Settles the second half of OQ-19; extends ADR-017.
- **Consequences.** The contract-expression grammar gains variant tests (needed in Phase 1); the check runs at T0. Like the struct rule, it is mention-based, so a clause that mentions the variant without meaningfully constraining it still passes — the human remains responsible for contract quality.
- **Alternatives considered.** Defer until after v1 — rejected: leaves the main gaming route open. A semantic check that the variant is determined by the inputs — rejected for v1: verifier cost; mention-based matches ADR-017.

### ADR-027: Future concurrency, if any, is share-nothing
- **Decision.** Multi-threading stays a v1 non-goal. If concurrency is added later, it will be share-nothing (message passing between isolated heaps), so reference counts stay non-atomic.
- **Context.** External review (2025-12-20, §3): moving to atomic counts later is a measurable slowdown, as Swift and Lean have found; deciding the direction now keeps v1 choices compatible.
- **Consequences.** No v1 work. Any later shared-memory proposal must supersede this ADR explicitly.
- **Alternatives considered.** Shared-memory threads with atomic counts — rejected: run-time cost on every count operation. Leave undecided — rejected: v1 runtime and ABI choices would not be checked against a direction.

### ADR-028: Two supported host targets; Z3 and LLVM are pinned system installs
- **Decision.** v1 supports `x86_64-unknown-linux-gnu` and `aarch64-apple-darwin`. Z3 (major 4, minor ≥ 12) and LLVM 18 are installed from the platform package manager and linked through the `z3` and `inkwell` crates; C′ does not bundle or build either.
- **Context.** OQ-16 and OQ-17. The design point is a developer workstation or CI host (A-5); `proved` must be reproducible, so the solver version must be fixed per installation.
- **Consequences.** Every CI job runs on both targets. A machine without the required Z3 or LLVM cannot build C′. Solver upgrades are explicit version-pin changes.
- **Alternatives considered.** Bundle Z3 and LLVM — rejected: long builds and large artefacts. Add Windows in v1 — rejected: a third JIT/AOT calling-convention target and CI leg for no stated user.

### ADR-029: JSON-RPC and MCP share one stdio stream
- **Decision.** `cprimed` reads newline-delimited JSON-RPC 2.0 messages from standard input and writes replies to standard output. The same stream serves the C′ methods and the MCP methods. No socket is opened.
- **Context.** OQ-11. Every MCP client can launch a server over stdio; stdio restricts access to the parent program, which meets A-1 without authentication.
- **Consequences.** One client per daemon; requests run one at a time. Several sessions on one project run one daemon each; their writes are coordinated through the project write lock and conflict check (A-3).
- **Alternatives considered.** Unix domain socket — rejected for v1: needs access control and a discovery convention. Loopback TCP — rejected: reachable by every local user.

### ADR-030: `cprimed` re-synchronises sources before every request
- **Decision.** Before each request, `cprimed` compares every `.cprime`, `.opq` and `cprime.toml` file's modification time and size with the last-seen values, re-reads changed or new files into the Salsa inputs, and drops deleted ones.
- **Context.** OQ-12. Humans edit `.cprime` files outside `cprimed`.
- **Consequences.** No file-watching dependency; every request sees the files as they are when it starts. The scan is reported as `timings_us.sync`, outside the T0 budget. An edit that leaves both size and modification time unchanged is missed (see Risks).
- **Alternatives considered.** File watching — rejected: platform differences and missed events. Explicit reload call — rejected: easy to forget, stale checks follow.

### ADR-031 (amended by ADR-053): Accepted implementations carry an acceptance header; edited contracts make them stale
- **Decision.** Every accepted `.opq` begins with `;; cprime-accepted v1 context=<hash> evidence=<proved|tested> assumed=<ops>`. The context hash covers the spec and every definition its verification depended on. An implementation whose recorded hash differs from the current one is stale: `get_context` reports it and release builds refuse it. Stale implementations are not re-verified automatically.
- **Context.** OQ-13 and A-16. Without a record of what was verified, an edited contract silently leaves old code marked as accepted.
- **Consequences.** Evidence level and assumed provider operations are now persisted (amends A-16). Editing a callee's contract marks its callers stale. Re-verification is an agent action (`submit_impl`).
- **Alternatives considered.** Re-verify on every change — rejected: unbounded work on each edit. Ignore until resubmitted — rejected: stale code would reach release builds.
- **Amended.** The header records the implementation's own evidence only. Effective evidence across callees is computed at report time and never stored (ADR-053).

### ADR-032: The runtime build produces the project's own `cprimed`; `cprimec daemon` launches it
- **Decision.** The project-specific runtime build generates one Cargo package that produces two artefacts: the project's `cprimed` executable (the `cprime-daemon` library plus every provider, statically linked) and a static library of the runtime and providers for release linking. `cprimec daemon` checks a runtime fingerprint, rebuilds when stale, then replaces itself with the project's `cprimed`.
- **Context.** OQ-22. JIT code must call third-party providers in-process; dynamic loading is a non-goal, so the providers must be linked into the daemon binary itself.
- **Consequences.** There is no generic `cprimed` binary. The first launch, and the first launch after a provider change, runs a Cargo build. Development and release link the same provider code.
- **Alternatives considered.** Load the runtime as a shared library into a generic `cprimed` — rejected: dynamic loading is a non-goal. Refuse to start when stale — rejected: an agent host cannot run the manual step.

### ADR-033 (amended by ADR-053): Verification is modular; every call site proves the callee's `requires`
- **Decision.** A call from one implementation to another spec is verified against the callee's contract: the caller proves the callee's `requires` at each call site (a call-precondition obligation) and assumes its `ensures`. Callee bodies are never inlined into the caller's proof.
- **Context.** The HLD lists obligations for `ensures`, trap sites and `decreases` only. Without call preconditions, a caller could invoke a callee outside its contract and rely on `ensures` facts that do not hold.
- **Consequences.** Proof cost per submission stays proportional to the candidate, not its call tree. A callee can be specified before it is implemented. Editing a callee's contract makes callers stale (ADR-031).
- **Alternatives considered.** Inline callee bodies — rejected: proof size grows with the call tree and recursion cannot be inlined. Trust callers — rejected: unsound.
- **Amended.** Because a caller assumes its callees' contracts, its effective evidence is the weakest across its callees' implementations, reported with every accepted result (ADR-053).

### ADR-034: Post-v1 directions — expressiveness parity, not feature parity
- **Decision.** After v1, C′ aims for expressiveness parity with Rust 2024: every program a Rust 2024 developer writes has a C′ formulation that keeps the four guarantees (type safety, effect containment, contract satisfaction at the evidence level the `assurance` policy demands, termination unless `Diverge` is granted) and meets the release performance target (within 2× hand-written Rust, G10). C′ may reach a program through a different construct. Every Rust 2024 capability absent from v1 carries exactly one post-v1 direction: **Planned** (a construct and its order), **Substitute** (an existing C′ construct that expresses the same programs), **Revisit on evidence** (a named measurement decides) or **Excluded** (refused, with a substitute named). The gap inventory, rows 1–24, is held in RFC 01 and decided by ADR-035 to ADR-042; rows 23 and 24 are decided here: panics and `catch_unwind` → **Substitute** (traps proved unreachable, ADR-006; recoverable failure through `Result`); files, network and launching other programs → **Substitute** (user-supplied providers, ADR-021). v1 non-goals that are not Rust 2024 capabilities — user-defined effect handlers in `.opq`, capability tokens as runtime values, side-channel resistance — are **Revisit on evidence**: a program in the expressiveness corpus (P1) that cannot be written without them. Guarantees about behaviour the contract does not state are **Excluded** by definition.
- **Context.** RFC 01 (2025-12-20). The user asked for v1 to be comparable to the Rust 2024 compiler, then narrowed the goal to "same programs, not feature parity". Only ADR-027 recorded a post-v1 direction; every other non-goal was silent, so a v1 default, hash format or unreserved word could make a later feature a breaking change.
- **Consequences.** Every non-goal in `00-overview.md` names its direction. Planned features enter in dependency order: generic specs before interfaces (ADR-036), function values before closures (ADR-037). Parallel map is in v1 (ADR-044) and tasks follow it (ADR-040). No direction may change the meaning of a v1 source, a v1 `DefId` or a v1 acceptance header (ADR-043). An Excluded direction is reopened only by an ADR that supersedes the excluding one.
- **Alternatives considered.** Feature parity with Rust 2024 — rejected by the user: a borrow checker, shared-memory threads and aliasable mutation each break a guarantee or principle 3. Leave post-v1 undecided — rejected: v1 choices would not be checked against any direction.

### ADR-035: Modules, visibility and packages after v1
- **Decision.** **Planned.** (1) `.cprime` gains explicit `use` imports; `.opq` keeps the SA-11 resolution order and gains no imports, so implementations spend no tokens on them. (2) Visibility is an opt-in `priv` marker on a `.cprime` definition; the default stays public. A private definition is visible only inside its own module. (3) After (1), C′ packages let one project depend on another; a dependency is pinned by the `DefId`s it exports.
- **Context.** RFC 01 rows 1–2. v1 has no imports and no visibility; every definition is public (SA-11, OQ-29).
- **Consequences.** A v1 project needs no edit when imports arrive, because v1 resolution remains valid. Visibility is not part of a definition's meaning, so it is excluded from canonical serialisation and changes no `DefId`. A private definition is still verified modularly (ADR-033).
- **v1 compatibility.** Public by default keeps v1 sources meaning the same; `use` and `priv` are future-reserved words in `.cprime` (ADR-043).
- **Alternatives considered.** Private by default (Rust's choice) — rejected: every v1 definition would become invisible outside its module. Imports in `.opq` as well — rejected: costs implementation tokens (H1) for no expressive gain over SA-11 resolution.

### ADR-036: Generic specs and interfaces after v1
- **Decision.** **Planned**, in this order. (1) A `spec` may declare type parameters; every instance is monomorphised before IR lowering, as type definitions are (ADR-015), and verified per instance. (2) After (1), an `interface` declared in `.cprime` names a set of operation signatures over a type parameter plus laws written as contract expressions; an `impl` of an interface for a type is written in `.opq`, one operation per file, and each law becomes a proof obligation of every operation it mentions. Dispatch is static only. `dyn Trait` is a **Substitute**: a sum type over the implementing types, or a record of spec-typed function values (ADR-037).
- **Context.** RFC 01 rows 3, 6 and 7. v1 type definitions may be generic, but `SpecDef` has no type parameters (`cprime-core`), and the design has no traits or typeclasses.
- **Consequences.** Laws make interface semantics checkable rather than nominal. Monomorphisation keeps the IR free of type parameters. A generic spec with *k* instances produces *k* verification runs; the latency budget applies per instance.
- **v1 compatibility.** Generic specs use a new canonical serialisation tag rather than changing the v1 spec tag `0x40` (ADR-043); `interface` and `impl` are future-reserved words in `.cprime`.
- **Alternatives considered.** Dynamic dispatch through vtables — rejected: an indirect call has no single contract to verify against, which breaks ADR-033. Interfaces without laws — rejected: an implementation could satisfy the signatures and nothing else.

### ADR-037: Function values and closures after v1
- **Decision.** **Planned**, in this order. (1) Non-capturing function values: a parameter whose type names a spec (a *spec-typed parameter*) accepts any spec or implementation with the same signature, effect set and a contract that implies the named spec's contract. A call through it is modelled by the named spec's contract, so modular verification (ADR-033) holds unchanged. (2) After (1), closures that capture existing values only; a closure that refers to itself, directly or through a binding it is stored in, is rejected.
- **Context.** RFC 01 rows 4–5. v1 has no first-class functions; a function name appears only in call position (SA-12, OQ-30). `Type::Fn` already carries an effect set and `Diverge` already propagates through it (ADR-024).
- **Consequences.** Termination: a call through a spec-typed parameter is treated as a call to every definition passed as a value of that spec type in the project, and those edges join the strongly connected component computation of ADR-024; the component's shared measure must then decrease. Acyclicity (ADR-023): a non-capturing value references no heap value, and a closure built from existing values cannot reference itself.
- **v1 compatibility.** None needed beyond ADR-043: SA-12 forbids function values in v1, so no v1 source uses one; `fn` becomes a future-reserved word in `.opq`.
- **Alternatives considered.** Function values typed by bare signature — rejected: a call through one would have no contract, so callers could assume nothing about the result. Unrestricted recursive closures — rejected: they tie a knot and break ADR-023.

### ADR-038 (amended by ADR-051): Iteration — recursion stays; loops on evidence; tail calls after v1
- **Decision.** (1) Recursion with `decreases` remains the iteration construct (ADR-024); it expresses every Rust loop. A `loop` form with mandatory `invariant` and `decreases` clauses is **Revisit on evidence**: it is added only if the H1 measurement (G9) or the P1 corpus shows loops cut model tokens by at least 10% or raise first-submission acceptance. (2) Local mutable variables are a **Substitute**: a new `let` binding of the same name lowers to the same SSA. (3) **Planned:** a tail call — a call whose result is returned directly with no reference-counting work after it — compiles to a jump, does not increment the call-depth counter, and still consumes fuel.
- **Context.** RFC 01 rows 8–9 and OQ-3. In v1 every function entry increments `rt.depth` and traps `StackLimit` above `limits.max_call_depth` (1,000 … 10,000,000; default 100,000), so iteration by recursion is bounded by that limit. Inferred borrowing (ADR-022) can leave reference-counting work after a call in tail position.
- **Consequences.** Until (3) ships, a v1 program iterating deeper than `max_call_depth` traps `StackLimit`. Shipping (3) is backward compatible: it removes traps and changes no successful result. The tail-call design must keep fuel accounting per function entry (`cprime-codegen` prologue) and decide how `cprime-memory` places drops so that a call stays in tail position.
- **v1 compatibility.** `loop` and `invariant` are future-reserved words in `.opq` (ADR-043). v1 makes no promise about stack use for tail calls, so (3) breaks nothing.
- **Alternatives considered.** Tail calls in v1 — deferred: changes `cprime-codegen`, `cprime-memory` and the interpreter in the middle of the v1 plan. Loops now — rejected: adds loop invariants to the verifier with no evidence that they help agents (ADR-024).
- **Amended.** Servers do not iterate in C′ code: the runtime spawns a task per request (ADR-051), so the call-depth limit applies per task and tail calls remain post-v1.

### ADR-039: Permanent exclusions — mutable references, interior mutability, ownership annotations
- **Decision.** **Excluded**, permanently: (1) mutable references (`&mut`) and interior mutability (`Cell`, `RefCell`); (2) reference cycles (`Rc` cycles, `Weak`) and a cycle collector; (3) a borrow checker, lifetimes and any other ownership annotation written by agents. Substitutes: (1) pass a value and return the updated value; FBIP reuses the memory in place when the count is 1 (ADR-002). (2) The arena pattern — a `Vec` of nodes addressed by `U64` index — expresses graphs and doubly-linked lists; index bounds are checked and proved (ADR-006). (3) Inferred borrowing (ADR-022) closes the performance gap first; region inference is **Revisit on evidence** after the Phase 5 benchmarks (G10).
- **Context.** RFC 01 rows 10–12; confirmed by the user on 2025-12-20. Immutability is what makes values acyclic (ADR-023), lets Z3 reason about values without modelling a mutable heap, and keeps principle 3 (agents never annotate ownership).
- **Consequences.** Some Rust programs take a different shape in C′: mutation becomes value passing, pointer graphs become arenas. The release performance of those shapes is measured by G10 and P1. Reopening any of the three requires an ADR that supersedes this one.
- **Alternatives considered.** Mutable references with a borrow checker — rejected: principle 3. Mutable references with a cycle collector — rejected: breaks ADR-023 and adds run-time cost to every program. Interior mutability restricted to one task — rejected: still aliasable mutation, which the verifier would have to model as a heap.

### ADR-040 (amended by ADR-051): Concurrency after v1 — share-nothing tasks
- **Decision.** Implements the direction of ADR-027, which stands. (1) **Excluded**, permanently: shared-memory threads, `Arc`, `Mutex` and atomics in C′ values. (2) **v1** (amended by RFC 02, ADR-044): a parallel map of one spec over a `Vec`, run across worker threads in release binaries, with inputs copied into worker heaps. (3) **Planned after v1, next in line:** share-nothing tasks, each with its own heap. A message is deep-copied into the receiving heap, or moved without a copy when its count is 1. Spawning a task and sending or receiving on a channel are operations of a built-in capability, so the effect checker covers them; a task that never ends needs `Diverge`. Work starts when the first program in the P1 expressiveness corpus needs tasks. (4) `async`/`await` is a **Substitute**: tasks with blocking provider operations; no function colouring.
- **Context.** RFC 01 rows 13–16; the shared-memory exclusion confirmed by the user on 2025-12-20. ADR-027 fixed share-nothing so that reference counts stay non-atomic. RFC 02 (2025-12-20): the user chose parallel map in v1 and tasks after v1.
- **Consequences.** Counts stay non-atomic in every heap. A message costs a copy unless it is uniquely owned. Termination of a program of tasks is argued per task; deadlock freedom is not a v1 or post-v1 guarantee and must be stated as a contract where a program needs it.
- **v1 compatibility.** None needed: concurrency adds a capability and a runtime, not a change to v1 constructs.
- **Alternatives considered.** Shared-memory threads with atomic counts — rejected (ADR-027). A language-level `async` — rejected: splits every library into blocking and non-blocking versions, with no expressive gain over tasks.
- **Amended.** v1 also runs share-nothing tasks, but only the runtime spawns them, one per server request (ADR-051). Item (3), program-spawned tasks with channels, stays next in line after v1 and reuses ADR-051's task heap and failure isolation.

### ADR-041: Foreign code, dynamic loading and macros stay out of `.opq`
- **Decision.** **Excluded**, permanently, from `.opq`: `unsafe`, FFI, raw pointers, inline assembly, dynamic loading of code, and macros (`macro_rules!`, procedural macros). Substitutes: foreign code runs only inside statically linked providers (ADR-021); agents write the expanded code that a macro would generate. **Planned:** provider-supplied contracts (OQ-27), fuzz-checked when a provider is bound, so that `proved` evidence covers calls into providers rather than assuming them.
- **Context.** RFC 01 rows 17–18; the foreign-code exclusion confirmed by the user on 2025-12-20. The effect checker cannot see inside foreign code (ADR-005), and verification needs the code it checks in expanded form.
- **Consequences.** The trusted computing base stays the compiler plus the providers a project links. Code generation by macro is replaced by code generation by the agent, which `cprimed` then verifies like any other implementation.
- **v1 compatibility.** None needed: v1 already excludes all of these.
- **Alternatives considered.** `unsafe` blocks with a proof obligation — rejected: the verifier has no model of raw memory. Hygienic macros in `.cprime` — rejected: contracts must be readable by humans as written.

### ADR-042: Primitive types, text and collections after v1
- **Decision.** **Planned.** (1) Integer types `U16`, `U32`, `U128`, `I8`, `I16`, `I32`, `I128`, with checked arithmetic (ADR-006); C′ has no `usize`, `U64` serves that role. (2) Floating point `F32` and `F64` with IEEE 754 semantics; a spec whose signature or contract uses them defaults to `assurance tested`, and may still request `proved` and accept `VerificationInconclusive`. (3) Text: `Str`, valid UTF-8 by construction, and `Char`, a Unicode scalar value; the Value JSON encoding (SA-17) gains a JSON string form for `Str`. (4) Persistent maps and sets in the prelude, specified by an SMT model (arrays or sequences) and implemented by the runtime. Every addition keeps one semantics shared by the SMT encoder and the fuzzing evaluator (SA-19).
- **Context.** RFC 01 rows 19–22. v1 has `Unit`, `Bool`, `U8`, `U64`, `I64`, `Bytes`, `FixedBytes`, `Vec`, structs and sums (SA-22, SA-23). Z3 bit-vectors extend to other widths without new theory; Z3 floating-point reasoning is expected to exceed the T2 budget (≤ 250 ms per obligation) on typical obligations — unmeasured.
- **Consequences.** Floating-point code reaches `tested` evidence by default, not `proved`. Each new type adds a row to the Value JSON encoding and to the provider boundary types (ADR-021) only by a later decision.
- **v1 compatibility.** The new type names are reserved type names in v1 (ADR-043), so no v1 definition can take a name that a future built-in will claim.
- **Alternatives considered.** Fixed-point arithmetic instead of floats — rejected: Rust programs that use `f64` would not port. `Str` as an alias of `Bytes` — rejected: UTF-8 validity would become a contract every spec restates.

### ADR-043 (amended by ADR-050): v1 compatibility hooks for post-v1 features
- **Decision.** v1 adopts three rules so that the Planned directions of ADR-035 to ADR-042 stay additive. (1) **Future-reserved words** are lexed as reserved and rejected wherever a name is expected: `.cprime` — `use`, `priv`, `interface`, `impl`, `func` (ADR-050); `.opq` — `fn`, `loop`, `invariant`. (2) **Reserved type names** cannot be declared as a type, capability or generic parameter: `U16`, `U32`, `U128`, `I8`, `I16`, `I32`, `I128`, `F32`, `F64`, `Str`, `Char`. (3) **Additive canonical serialisation:** a post-v1 construct is encoded under a new tag byte; the byte layout of every v1 tag never changes, and attributes that do not change a definition's meaning (visibility) are excluded. The hash domain tag is `cprime.def.v1` (ADR-050) and does not change after v1.
- **Context.** RFC 01 acceptance criteria 6 and 8. A `DefId` is the hash of canonical serialisation, and changing serialisation changes every hash (ADR-004), which would make every acceptance header stale (ADR-031). Name resolution tries prelude and built-in names before absolute paths (SA-11), so a new built-in type could capture a reference that resolved elsewhere in v1.
- **Consequences.** Post-v1 features add tags and words; they never re-hash v1 definitions or re-parse v1 sources differently. v1 users lose seven words and eleven type names as identifiers.
- **Alternatives considered.** A language-version marker in each source file — rejected: adds a line to every file for a problem reservation removes. Bump the hash tag when features arrive — rejected: invalidates every acceptance header in every project.

### ADR-044 (amended by ADR-051): Parallel map in v1
- **Decision.** The `.opq` form `(par_map S xs)` applies spec `S` to every element of `xs : Vec[T]` and yields `Vec[U]`, where `S` takes one `T` and returns `U`. `S` has no `requires` clause, does not use `Diverge`, and uses only capabilities whose operations are all `deterministic` and whose bound provider is thread-safe (ADR-045). Its meaning is the sequential map, everywhere. Only release binaries run it in parallel: the input is split into contiguous chunks over up to `limits.workers` worker threads, each with its own worker context and heap. Each element is copied into the worker heap by a memoised deep copy that preserves sharing within the element and never touches the caller's counts. Results are copied back in index order after the workers finish. A trap in a worker records its element index and parks the thread; the main thread reports the lowest recorded index. A `par_map` inside a worker runs in sequence. `cprimed` (interpreter, JIT, fuzzing) always runs it in sequence.
- **Context.** RFC 02 (2025-12-20). Release programs must be able to use several cores (G10 compares with Rust, which can). Reference counts must stay non-atomic (ADR-027), there are no function values (SA-12), and AOT code is compiled without trap checks, so a worker cannot return early from a trap.
- **Consequences.** Verification and fuzzing are unchanged: the form is the sequential map, `S` has no call precondition, and facts about result elements come from `S`'s `ensures`. A release result never depends on the worker count. When the only traps are ordinary ones (overflow, division by zero, index, unreachable, provider panic), the reported trap is the same on every run. Resource limits may be hit by a different element from run to run. Each element costs a copy in and a copy out. `par_map` is an `.opq` reserved word with a v1 meaning and gets canonical serialisation tag `0x8C` (SA-50). `RtContext` keeps its fields; `mode` gains the value 2 (worker).
- **Alternatives considered.** Parallel execution inside `cprimed` too — rejected: checking gains nothing, and fuel, depth and leak accounting would need to become thread-aware. Unwinding a worker on a trap — rejected: needs trap checks in AOT code on every trap site. Sharing the caller's values with workers read-only — rejected: a worker's reference-count operations on shared children would race. Fork-join of different specs — deferred: not requested.
- **Amended.** The deterministic-operations rule applies to `par_map` only. Server handlers (ADR-051) may use non-deterministic operations of thread-safe providers, because a handler's result is not combined across workers.

### ADR-045 (amended by ADR-051): Thread-safe providers
- **Decision.** A provider declares `thread_safe: true` in its `ProviderDecl` when every one of its operations may be called concurrently from several threads, each with its own `RtContext`. Default `false`. Only capabilities bound to a thread-safe provider, and whose operations are all `deterministic`, may be used by a spec mapped with `par_map` (ADR-044). `host::Crypto` declares `true`; `host::Clock`, `host::Random` and `host::Log` declare `false`.
- **Context.** RFC 02 OQ-1, option (b), chosen by the user on 2025-12-20: parallel hashing and signature checking need a provider operation. Provider operations have no receiver (`OpFn`), so their state can live only in Rust statics, which Rust's compiler forces to be thread-safe unless the provider uses `unsafe`.
- **Consequences.** The flag is a provider's claim, like `deterministic`. It joins the trusted computing base and is not checked by C′. Rust's `Sync` rule enforces most of it; `unsafe` code or `static mut` can break it. A provider that does not declare it cannot be used inside `par_map`, and rejection names the capability.
- **Alternatives considered.** No capabilities inside `par_map` (option a) — rejected by the user: excludes the main use case. Infer thread safety — rejected: C′ cannot inspect provider code. Serialise all provider calls behind one lock — rejected: removes the speed-up for exactly the work that needs it.
- **Amended.** `host::Clock`, `host::Random` and `host::Log` declare `thread_safe: true` (ADR-051), with the implementation rules in `cprime-host-providers`. A transport provider must declare `thread_safe: true`.

### ADR-046: A person approves every contract; an agent may draft it
- **Decision.** A `.cprime` file is the human-approved surface, not necessarily the human-typed one. A person may write it, or an AI agent may draft it from a plan the person has already agreed, in a spec-driven process such as CSPA. Either way, a person reviews the contract against the plan and approves it before any `.opq` is implemented against it, and the approval is recorded outside C′, in version control, by the person (as with accepted `.opq`, A-12). `cprimed` does not distinguish who wrote a `.cprime` file and gains no request for writing one: an agent drafts contracts with its host's ordinary file tools, and `cprimed` picks the edit up before the next request (ADR-030).
- **Context.** Amends ADR-001, which assigned `.cprime` to humans as authors. Teams already working spec-driven have agents produce artefacts from an agreed plan and keep a person as the reviewer. The value of the split in ADR-001 is that the artefact a person must check is short and declarative: data, permitted capabilities, `requires` and `ensures`, with no control flow, memory handling or hidden calls. That makes review cheaper and more reliable than reading an implementation, whoever typed the contract.
- **Consequences.** The human gate is contract *approval*, not contract typing. C′ checks an implementation against its contract; it does not check a contract against the plan it was drafted from, so that review is the only safeguard for that step and C′ cannot enforce it. An agent can write `.cprime` files and implement against them in one session without approval; preventing that is the process's job (for example, CSPA requires every change to shared state to be approved by a person). Contract completeness still carries the whole safety argument (ADR-001); `UnconstrainedOutput` and the completeness rule (ADR-026) help the reviewer see gaps whoever drafted the contract.
- **Alternatives considered.** Agents never write `.cprime` *(drafted)* — rejected: `cprimed` cannot tell who edited a file on disk, so the rule could not be enforced, and it would forbid an established way of working. An approval marker in the `.cprime` file, checked by `cprimed` before accepting implementations *(drafted)* — rejected for v1: with no authentication (A-1) the agent could write the marker itself, so it would prove nothing. A `cprimed` request for drafting contracts *(drafted)* — rejected: it would add an interface with no checking benefit over writing the file.

### ADR-047 (amended by ADR-055): Witness traces on counterexample rejections
- **Decision.** When T2 or T3 rejects a submission with a counterexample (`ContractViolation` or `ReachableTrap`), `cprimed` replays the counterexample through the IR interpreter and attaches a **witness trace** to the diagnostic: the executed steps that the failing clause or trap depends on, in execution order, each with its source node, span, kind and value. For a T2 counterexample, the replay takes from the solver's model only the values the SMT encoding leaves free: the results of the candidate's spec calls, provider operations and parallel maps. It computes everything else, and it never enters a callee. For a T3 counterexample, the replay runs callees for real and serves stubbed operations from a log recorded during the violating JIT call. Each trace states whether it reproduced the reported outcome (`reproduced`), stopped at a trap reached before it (`trap_first`), or disagreed with it (`not_reproduced`). Values the solver chose, the fuzzer generated or a provider returned are marked as such. A trace is bounded by `verify.witness_max_steps` (default 64); 0 turns traces off.
- **Context.** RFC 03 (2025-12-20). An agent troubleshoots by reading source and outcomes, not by stepping a debugger. A counterexample gives inputs, expected and actual values but not the path between them, so the agent re-derives the path by reading the implementation. The SMT encoding computes every instruction exactly except at four kinds of free value, so a replay that supplies those values follows the equations Z3 solved.
- **Consequences.** A rejection's diagnostics grow by at most `witness_max_steps` steps each. Acceptance is unchanged: a trace never turns a rejection into an acceptance (ADR-025), and verification runs on IR without tracing. A `not_reproduced` trace can only mean a defect in the trusted computing base, so replay also checks the SMT encoder against the interpreter. Witness time is reported separately from T2 and T3. The interpreter gains a replay entry point; its existing `run` is unchanged. `node_map` gains an entry for every non-constant value lowered from source. Whether traces help agents is measured on the H1 corpus and reported either way (RFC 03 criterion 17).
- **Alternatives considered.** Read the trace straight from the Z3 model, which binds every IR value — rejected: values at undefined operations (division by zero, out-of-range index) are arbitrary in the model and would be shown as if computed, and the TCB cross-check would be lost. Step into accepted callees — rejected: verification is modular, so the callee's body is not what the proof relied on. Return the whole executed path — rejected: token cost grows with implementation size; the dependency slice is what explains the failure. A separate `explain` method — deferred: a new daemon method is outside RFC 03.
- **Amended.** `reproduced`, `trap_first` and `not_reproduced` keep their meaning for the model replay, which checks the encoder against the interpreter. Whether the counterexample can happen in the running program is reported separately as `concrete` (ADR-055).

### ADR-048: Release trap locations; call chain only in opt-in builds
- **Decision.** Every release binary reports where it stopped. A trap (exit 3) names the qualified spec and the `file:line:column` of the trapping expression in its accepted `.opq` file. A call-depth limit (exit 7) names the spec whose entry exceeded it. A provider panic (exit 6) names the capability operation and the location of the `perform`. Each trap edge in AOT code passes a constant site id to the runtime, and a table generated into the binary maps site ids to names and locations. A release build made with `release.call_chain = true` also reports, on exits 3 to 7, the chain of spec calls active at the moment of failure, outermost first. This includes the CPU-time (4) and memory (5) limits, which have no location of their own. Exit codes and the leading text of every stderr line stay as they were. No release output contains a program value.
- **Context.** RFC 03 (2025-12-20), OQ-1 resolved by the user on 2025-12-20: option (a). A location costs nothing on the normal path because it travels only on the cold trap edge. A call chain needs a store on every function entry, which counts against G10.
- **Consequences.** `RtContext` gains two trailing fields, `site` and `chain`, at offsets 56 and 64; existing offsets do not change. Each `perform` in AOT code stores its site id first: one store per provider call, measured in the Phase 5 benchmarks with the rest of the overhead. AOT code checks `slice` bounds inline so that this trap has a site. Default builds report exits 4 and 5 without a location. Inside `par_map`, the report names the lowest-index trapping element (ADR-044). The AOT-vs-interpreter differential test compares trap locations as well as trap kinds.
- **Alternatives considered.** Call chain always on (option b) — rejected by the user: a cost on every call in every program. Both opt-in (option c) — rejected: the location costs nothing and every crash report needs it. DWARF debug information with an unwinder — rejected: needs symbolisation at crash time, which the async-signal-safe exit path cannot do, and is far larger. Printing input values — rejected: release inputs are user data.

### ADR-049 (amended by ADR-050, ADR-056): The language is C′; contracts are pacts
- **Decision.** The language and its toolchain are named **C′** ("C-prime"; `C'` where only ASCII is possible; `cprime` in identifiers). Toolchain identifiers use the `cprime` stem: binaries `cprimec` and `cprimed`, crates `cprime-*` and Rust paths `cprime_*`, MCP tools `cprime_get_context`, `cprime_check_impl`, `cprime_submit_impl`, log targets, runtime ABI symbols (`cprime_rt_*`, `cprime_fn_<hex>`, `cprime_entry`), the provider convention `fn cprime_provider()`, the generated package `cprime-runtime-<name>`, the inventory format `cprime-inventory-v1`, the configuration key `runtime.cprime_source`, the database type `CprimeDatabase`, and the release stderr prefix `cprime:`. The contract layer keeps its name: a contract is a **pact**, contract files are `.pact`, the manifest is `pact.toml`, project state lives in `.pact/`, and the hash domain tags (`pact.def.v1`, `pact.mono.v1`, `pact.ctx.v1`) and the acceptance header (`;; pact-accepted v1`) are unchanged. The agent surface stays `.opq`. Earlier ADRs use the new names; their decisions are unchanged.
- **Context.** RFC 04 (2025-12-20). "Pact" collides with Kadena's Pact, a smart-contract language verified with Z3, and with Pact (pact.io), a contract-testing tool that holds the `pact` crate; `pact-cli` is also taken on crates.io. The user chose C′ on 2025-12-20: the name is for the surface people read and approve, it signals an evolution over C and Go, and in Hoare-logic notation the prime marks the post-state that `ensures` describes. The only prior C′, a SourceForge project, has been dormant since 2013.
- **Consequences.** No behaviour changes. Every `DefId` and acceptance header keeps its v1 meaning (ADR-043) because the hash tags and header are unchanged. ADR-048's "leading text unchanged" applies with the prefix `cprime:`. The HLD folder `docs/design/pact/` and the spec-set id "pact" stay as internal topic ids. The repository name, a trademark search and the logo artwork are outside the specification.
- **Alternatives considered.** Keep "Pact" — rejected by the user: two live collisions in the same space. Proviso and Avow (meaning-first names) — not chosen. C\* and C⊢ — C\* is taken (Thinking Machines, 1987); C⊢ cannot be typed or said. Rename the contract layer too (`.cprime` files) — rejected: "the language is C′, the contracts are pacts" keeps the existing vocabulary and avoids touching ADR-043.
- **Amended.** ADR-050 moves the contract layer to the `cprime` stem and the `.cprime` extension; the rest of this decision stands.
- **Amended.** The HLD folder and the spec-set id no longer keep the old name: they are `docs/design/cprime/` and "cprime" (ADR-056).

### ADR-050 (amended by ADR-056): Contracts use a Go-like syntax in `.cprime` files
- **Decision.** Contract files use the extension `.cprime` and a concrete syntax in the C and Go family. *Layout:* every declaration with a body (`spec`, `entry`, `struct`, `enum`, `capability`) puts it in braces; indentation never affects meaning; a newline ends a statement, following Go's semicolon-insertion rule; each clause is one line that starts with its keyword, in the fixed order `uses`, `assurance`, `unconstrained`, `requires`, `ensures`, with `requires` and `ensures` lines repeated as consecutive lines. *Declarations:* a name comes before its type (`user_id U64`), and consecutive names may share a type (`key, msg Bytes`); there is no `:` and no `->`; the return type follows the parameters, and no return type means `Unit`; `struct S { … }` and `enum S { … }` declare records and sums, and `type` declares aliases only; generics use square brackets (`Result[T, E]`, `FixedBytes[32]`); capability operations carry no `fn`; `ensures` always names the result `result`, and a parameter called `result` is an error; `assurance proved` is the default and is left out by convention. *Contracts:* operators and precedence are unchanged; `::` remains the path separator; byte literals are `x"…"` (`.opq` keeps `0x…`). *Reserved words:* `.cprime` reserves `struct` and `enum`, future-reserves `func`, and no longer reserves `fn`. *Names:* the contract layer moves to the `cprime` stem — `.pact` → `.cprime`, `.pact/` → `.cprime/`, `pact.toml` → `cprime.toml`, `pact.def.v1`/`pact.mono.v1`/`pact.ctx.v1`/`pact.runtime.v1` → `cprime.def.v1`/`cprime.mono.v1`/`cprime.ctx.v1`/`cprime.runtime.v1`, `;; pact-accepted v1` → `;; cprime-accepted v1`, `libpact_runtime.a` → `libcprime_runtime.a`, `target/pact/` → `target/cprime/`. A contract is called a *contract*. The HLD folder `docs/design/pact/` and the spec-set id "pact" are kept as internal topic ids. The `.opq` surface is unchanged.
- **Context.** RFC 05 (2025-12-20), decided by the user. A person approves every contract (ADR-046), so the contract surface is optimised for reading. ADR-049 named the language for "an evolution over C and Go", yet the `.pact` syntax followed Rust (`fn`, `->`, `name: Type`, `A | B`). On the concept's two specs, the new syntax uses 160 tokens instead of 196 and 62 punctuation tokens instead of 94, with the same words; the counting method splits text on `==>|::|->|==|<=|>=|\+\+|&&|\|\||[A-Za-z_][A-Za-z0-9_]*|\d+|\S` and counts tokens that do not start with a letter, digit or `_` as punctuation. GitHub Linguist maps `.pact` to Kadena's Pact and `.cp` to C++ and Component Pascal, and maps `.cprime` to nothing. Nothing is implemented and no v1 source exists, so the change breaks no user (ADR-043 binds v1 sources once v1 ships).
- **Consequences.** Only the parser changes: it produces the same untyped syntax tree (`cprime_core::ast`), so type checking, effect checking, verification, the IR and the canonical serialisation layout are unchanged, and the syntax change alone leaves every `DefId` as it was, because the hash excludes source text and spans. The renamed domain tags change every `DefId` and the text of every acceptance header; no stored `DefId` or accepted file exists, so nothing is invalidated. `ContractExpr.text` carries the new clause text into diagnostics; the verifier's split at the top-level `==` still applies. `get_context` prelude lines are rewritten in the same style. Readability is unmeasured until hypothesis H2 is measured in Phase 3 (`10-risks.md`). ADR-036's reserved generic position becomes `[…]` after a spec name, where Go's constraint form `[T Ord]` will carry interface bounds.
- **Alternatives considered.** `.` as the path separator — rejected: ambiguous with field access, and would change the tree. C's type-first declarations — rejected: they read right to left once nested. Indentation-significant bodies — rejected: indentation would carry meaning, and a misplaced line could join the wrong declaration. Braces only on name-led bodies — rejected: two rules where one suffices. Grouped `requires { … }` blocks — rejected: a second form, and a changed line in a diff no longer shows its clause kind. A nameable result — rejected: no example uses it. `+` for concatenation — rejected: the parser would need types. `concat(…)` — rejected: changes the tree. Dropping the `module` line — rejected by the user. `.cp` — rejected: GCC and GitHub treat it as C++. `.cpr` — not chosen: it is the Cubase project format. Renaming only the extension — rejected by the user in favour of one stem everywhere.
- **Amended.** The HLD folder and the spec-set id no longer keep the old name: they are `docs/design/cprime/` and "cprime" (ADR-056).

### ADR-051: Servers in v1 — each request is a runtime-spawned, share-nothing task
- **Decision.** A v1 program may be a server instead of a batch entry, declared by a `server` item that names a handler spec and a transport provider.
  - **Transport.** Requests come from the transport, a project-supplied provider implementing the transport interface of the provider SDK, as flat records of boundary types. C′ ships a sample HTTP transport crate, which is not built in. The transport interface is versioned (v1 = 1) and reserves a non-blocking receive result, `Pending`, which v1 runtimes treat as a transport error.
  - **Tasks.** For each request, the release runtime spawns a short-lived, share-nothing **task** that runs the handler once on one of `limits.workers` worker threads, in its own **task heap**. Objects in a task heap are freed by reference counting as everywhere else; the heap's free-all operation is used only when a task ends abnormally.
  - **Handler rules.** A handler has one parameter and no `requires`, does not use `Diverge`, and may use only capabilities bound to thread-safe providers; operations need not be deterministic.
  - **Failure and limits.** A trap or a per-task limit (`limits.request_fuel`, `limits.max_call_depth`, the process memory limit) ends only that task, and so fails only its request. Server binaries have no process CPU-time limit.
  - **Shutdown.** `SIGTERM` and `SIGINT` stop the transport and drain running tasks within `limits.shutdown_grace_ms`.
  - **Checking.** `cprimed` checks handlers as ordinary specs and runs no server.
  - **Concurrency model.** No C′ value is shared between threads, and reference counts stay non-atomic (ADR-027 stands).
- **Context.** RFC 06 (2025-12-20). The user set servers as an early use case, required multi-threading to be first-class from v1 without slowing single-threaded code, chose option A over tasks with channels (option B), and accepted both RFC recommendations (a transport plug-in; structured requests). At adjudication, the user asked that the decisions stay on the path to option B. Hence three choices:
  - a request is a task the runtime spawns;
  - the task heap is a general per-task heap, not a scratch area;
  - the transport interface is versioned, with non-blocking receive reserved.

  In v1 a server written in C′ code would fail for three reasons:
  - its loop is recursion and traps at the call-depth limit (ADR-038);
  - `RLIMIT_CPU` caps every release process at 24 hours of CPU time (SA-21);
  - a release binary is one-shot (SA-16).
- **Consequences.**
  - Handlers get the full guarantees (types, effects, termination, contracts) per request, and contracts can state properties of requests and responses.
  - The dispatch loop, the threads and the transport are trusted runtime and provider code; every transport joins the trusted computing base (ADR-021).
  - A trap ends a task, not the process. New exit codes 8 (shutdown grace period expired) and 9 (transport failed to start).
  - The fuel check in every function prologue, which now bounds a task's work, is also the hook a future scheduler can use to pause a task.
  - **Path to option B (ADR-040 (3)), which stays next in line after v1:** programs will spawn their own long-lived tasks and connect them with channels. This reuses the task heap, failure isolation and thread-safe providers unchanged. Long-lived connections will need the reserved non-blocking transport mode and many tasks on few threads. Handlers do not change.
- **Alternatives considered.**
  - Tasks with channels in v1 (option B) — not chosen by the user: needs a scheduler and guaranteed tail calls in v1.
  - Shared-memory threads with atomic counts — rejected (ADR-027).
  - A built-in HTTP provider — rejected by the user: reverses the "no built-in network provider" non-goal.
  - Raw bytes to handlers — rejected by the user: leaves contracts almost nothing to say about requests.
  - Servers built from `par_map` — rejected: `par_map` takes a finite `Vec`, needs deterministic operations, and returns only when every element is done.
  - A reset-only scratch region per request — rejected at adjudication: long-lived tasks need a heap whose objects are freed individually.
  - An unversioned blocking-only transport interface — rejected at adjudication: a future scheduler could not adopt it without breaking transports.
  - A replacement thread after each trap instead of a recovery point — rejected: each parked thread keeps its stack, about 99 MiB at the default call-depth limit.

### ADR-052: Vacuous and inconsistent contracts are rejected before any obligation
- **Decision.** At T2, before any other obligation, `cprimed` checks that the spec's `requires` admits an input (V1) and that `requires` and `ensures` together admit an input–result pair (V2). An `unsat` check rejects every submission against the spec with `VacuousContract`, naming the clauses in the solver's unsat core; no other obligation runs. An `unknown` check never rejects: the reply carries a `VacuityUnknown` diagnostic of the new severity `note`. Decided results are cached per spec `DefId` for the life of the `cprimed` process.
- **Context.** RFC 07 (review point 2). Each obligation asks for an input that satisfies `requires` and violates a goal. When no input satisfies `requires`, every obligation is `unsat` and any implementation is accepted as `proved`. When the `ensures` clauses contradict one another, every implementation fails with a `ContractViolation` that points at the implementation, not the contract.
- **Consequences.** A vacuous contract can no longer yield `proved`. An inconsistent contract is reported as a contract defect. Diagnostics gain a non-error severity, and accepted replies may carry notes. The check costs two solver calls per spec revision per daemon, counted against the T2 budget. A contract satisfiable only in ways the human did not intend still passes; contract review (ADR-046) remains the safeguard.
- **Alternatives considered.** Warn instead of reject — rejected: no implementation can be meaningfully accepted against a contract with no satisfying input. Run the check in `check_impl` — rejected: T0–T1 has a 25 ms budget and no solver. Cache by the verification context hash — rejected: that hash also covers callees, which the check does not read, so it would re-run needlessly.

### ADR-053: Effective evidence is computed at report time, never persisted
- **Decision.** For an accepted implementation, `cprimed` and `cprimec` report its own evidence level and its effective level: the weakest evidence across every implementation it transitively calls, with the `tested` callees, the callees that are not current, and the provider operations assumed anywhere in that closure. The value is computed from each callee's current acceptance header whenever it is reported (`submit_impl`, `cprimec status`, release inventory) and is never stored.
- **Context.** RFC 07 (review point 3), OQ-1 resolved by the user on 2025-12-20: option (a). A caller assumes its callees' contracts (ADR-033), so a `proved` caller of a `tested` callee rests on fuzzing. The acceptance header records only the caller's own evidence, and storing callee evidence there would make every caller stale whenever a callee is re-accepted, and would change the `v1` header (ADR-043).
- **Consequences.** The header format and staleness rules are unchanged. Re-accepting a callee changes its callers' effective evidence immediately. A caller accepted before its callee is implemented reports effective `none`. The release refusal rule is unchanged; the release inventory now shows effective evidence.
- **Alternatives considered.** Persist in the caller's header (option b) — rejected by the user: re-acceptance would stale callers and the format would change. Both (option c) — rejected: two sources that can disagree.

### ADR-054: `deterministic` means deterministic and effect-free; providers are checked by known-answer vectors; provider calls are bounded during checking
- **Decision.** A provider operation marked `deterministic` must return equal results for equal arguments, perform no observable effect, and return without waiting on anything outside the call. It may appear in contracts, where T2 models it as an uninterpreted function and T3 evaluates it through the bound provider, through a memo shared with the program. Built-in `deterministic` operations ship known-answer vectors from their standards; `cprimec runtime build` runs every declared vector and refuses a runtime whose provider fails one. During checking, every call into provider code runs on a thread `cprimed` can abandon after `verify.provider_wait_ms`; an abandoned call yields an inconclusive or abstract result, never a hang.
- **Context.** RFC 07 (review points 4 and 7), OQ-2 resolved by the user on 2025-12-20: option (a); OQ-4 recommendation (abandon and mark). The flag meant only "same inputs, same result", so evaluating a contract at T3 could run provider code with an effect, nothing checked that an operation computed what its name says, and a provider call that never returned would hang `cprimed`.
- **Consequences.** The concept's contracts using `hmac_sha256` keep working. Vectors reduce, but do not remove, provider trust; providers stay in the TCB and are never described as verified. Third-party operations without vectors are allowed and reported. An abandoned thread is leaked until its provider returns; at most eight are kept per daemon.
- **Alternatives considered.** A separate logical model per operation in `.cprime` (option b) — rejected by the user: a larger language change overlapping the post-v1 abstract-model work. Forbid provider operations in contracts (option c) — rejected: breaks `generate_token`. Run fuzzing in a child process — rejected: heavier, and changes the JIT memory model.

### ADR-055: Every counterexample is classified concrete or abstract
- **Decision.** A `ContractViolation` or `ReachableTrap` with a counterexample carries `concrete`: `concrete` when the violation recurs with real `deterministic` providers and accepted callee implementations, `abstract` otherwise, with a reason (`no_violation`, `different_outcome`, `callee_unimplemented`, `stubbed_operation`, `resource_limit`, `provider_failure`, `provider_timeout`). For T2, a second replay of the counterexample, the concrete replay, decides it; for T3 the violating run decides it. Unsupported obligations get their own reason, `unsupported_construct`, distinct from `solver_unknown`.
- **Context.** RFC 07 (review point 8). The ADR-047 replay takes call and provider results from the solver's model, so it checks the encoder, not the program: a model may give `hmac_sha256` a value no provider returns, and the counterexample is `reproduced` yet cannot happen. An encoder gap and a solver `unknown` were reported identically.
- **Consequences.** An agent can tell a real bug from an artefact of modular or uninterpreted reasoning, and an inconclusive proof from an unsupported one. Concrete replay runs real `deterministic` providers (effect-free by ADR-054) and accepted callees, bounded by fuel, memory and `verify.provider_wait_ms`; its time is reported in `timings_us.witness`. Acceptance is unchanged: an abstract counterexample still rejects (ADR-025), because the contract must hold for every function with the operation's signature.
- **Alternatives considered.** Accept on an abstract counterexample — rejected: would make `proved` depend on provider code, contradicting ADR-033 and the uninterpreted model. Replace the model replay with the concrete one — rejected: loses the encoder cross-check of ADR-047. Omit `concrete` when the replay is impossible — rejected (RFC 07 OQ-5): the agent could not tell "not checked" from "not reported".

### ADR-056: Internal topic ids use the `cprime` stem
- **Decision.** The HLD folder is `docs/design/cprime/` and the spec-set id is "cprime". They replace `docs/design/pact/` and "pact", which ADR-049 and ADR-050 had kept as internal topic ids. Current documents use the new names; historical records (the changelog and earlier audit sections) keep the paths they were written with. Working documents under `docs/wip/` (RFCs and HLD-impact reviews) and `docs/outreach/` are local and no longer committed.
- **Context.** User decision, 2025-12-20, after RFC 07. The kept ids were the main remaining source of "Pact" in the repository, and a newcomer reading `docs/design/pact/` could take the project for the contract-testing tool or Kadena's Pact (ADR-049).
- **Consequences.** No behaviour, identifier, `DefId` or file format changes: the ids name documents, not anything the toolchain reads. `/spec`, `/breakdown` and `/sync-check` take the topic shorthand `cprime`. Paths quoted in historical records no longer resolve; they are records of what existed when they were written.
- **Alternatives considered.** Keep the ids (ADR-049, ADR-050) — rejected by the user. Rewrite the historical records too — rejected: they record decisions as made, and the commit history already holds every earlier path and every working document.
