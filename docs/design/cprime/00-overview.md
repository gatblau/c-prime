# C′ — Overview

> **Mode:** draft
> **Generated from:** inline context — `docs/cprime-concept.md` (Draft, revision 2)
> **Revision:** 0.12.0
> **Last updated:** 2025-12-20

## Summary

C′ is a verification-first systems programming language and compiler, written in Rust, for a workflow in which humans own the contracts and AI agents write the implementations. Humans write `.cprime` files, or approve ones an agent drafted from an agreed plan (ADR-046) (types, capabilities, behavioural contracts, termination and assurance policy); agents author `.opq` files (compact S-expression implementations bound to a `.cprime` spec). A long-running daemon, `cprimed`, type-checks, effect-checks, JIT-compiles and verifies each implementation against its contract (Z3 first, property-based fuzzing where policy allows) and persists only accepted implementations. A thin CLI, `cprimec`, builds the project-specific runtime and produces LLVM release binaries. Region inference, multi-threading, mutable references and FFI from `.opq` are out of scope for v1.

## Motivation

When an AI agent writes an implementation, a human currently has to review it line by line to trust it. That review is the bottleneck, and it scales poorly as the volume of generated code grows. C′ moves the human effort from reviewing implementations to writing or reviewing contracts, which are short and declarative: an implementation is accepted only when the compiler can show that it type-checks, performs no unauthorised effect, satisfies every `requires`/`ensures` clause at the evidence level the spec demands, and terminates (unless the spec grants `Diverge`).

Doing nothing leaves two options, neither acceptable for systems code: trust unreviewed generated code, or keep human line-by-line review as the gate. The language's guarantees stop at what the contract states. The design therefore also makes contract gaps visible (`UnconstrainedOutput`) instead of leaving them silent.

The project does not depend on hypothesis H1. If `.opq` turns out to save no tokens, verified acceptance of agent-written code is still the case for C′; H1 affects the shape of the implementation surface, not whether the project proceeds.

The concept does not name specific users, teams or incidents behind this motivation. That is recorded as OQ-20 (non-blocking).

## Goals

- **G1.** Parse, type-check and effect-check `.cprime` and `.opq` sources, including sum types, exhaustive `match`, monomorphisation and the termination check.
- **G2.** Lower implementations to C′-IR (block-argument SSA), insert Perceus reference-counting operations with FBIP reuse, and JIT-compile with Cranelift.
- **G3.** Discharge contracts with Z3. Where the spec's `assurance` is `tested`, fall back to property-based fuzzing when Z3 returns `unknown` or times out.
- **G4.** Serve `get_context`, `check_impl` and `submit_impl` over JSON-RPC 2.0, and as the MCP tools `cprime_get_context`, `cprime_check_impl` and `cprime_submit_impl`, from a long-running daemon.
- **G5.** Meet the latency budget: T0 ≤ 5 ms and T1 ≤ 20 ms p95 on a single implementation of at most 200 IR instructions with a warm Salsa database. T2 ≤ 250 ms per proof obligation and ≤ 2 s per submission. T3 is 10,000 iterations within ≤ 5 s.
- **G6.** Persist every accepted implementation atomically to its `.opq` file, so version control records each acceptance.
- **G7.** Let projects reuse existing Rust crates through a provider SDK (`cprime-provider`), statically linked into a project-specific runtime, with a provider inventory reported on every release build.
- **G8.** Produce AOT release builds through LLVM.
- **G9.** Measure hypothesis H1 (at least 40% fewer model tokens than idiomatic Rust, with no drop in first-submission acceptance rate) on the 50-task corpus, and report the result whether or not it holds.
- **G10.** Measure the release performance target (within 2× hand-written Rust run time on the Phase 5 benchmark suite), and report the result whether or not it is met.
- **G11.** Let release binaries spread work across CPU cores through `par_map` (ADR-044): results identical at every worker count, the memory limit holding across all workers, and the speed-up at 1, 2, 4 and 8 workers reported by the Phase 5 benchmark suite, whether or not it reaches 0.6 × the worker count.
- **G12.** Make failures explain themselves to the agent and to the person running a release binary (ADR-047, ADR-048). Every counterexample rejection carries a bounded witness trace, and every release trap names its spec and source location. Whether traces reduce the tokens and submissions an agent needs is measured on the H1 corpus and reported whether or not they do.
- **G13.** Let a release binary be a long-running server: for each request the runtime spawns a short-lived, share-nothing task that runs a handler spec on one of up to `limits.workers` worker threads; a failing request does not stop the server; and programs that are not servers run within 2% of their run time without the feature (ADR-051).

## Post-v1 goals

Not v1 deliverables; recorded so that v1 choices are checked against them (ADR-034).

- **P1.** Expressiveness parity with Rust 2024. Every program in the expressiveness corpus — the 50 H1 tasks (ADR-019) plus 20 Rust programs chosen for mutation-heavy, cyclic-data, concurrent, text and floating-point idioms — has an accepted C′ implementation, or maps to one gap-inventory row whose Planned direction has not shipped. Release builds of the accepted ones meet the G10 target. Report the result whether or not it holds.

## Non-goals

| Non-goal | Consequence for v1 | Post-v1 direction |
|---|---|---|
| Region inference | All heap values are reference counted. Revisit after the Phase 5 benchmarks. | Revisit on evidence (ADR-039, Phase 5 benchmarks against G10) |
| Concurrency beyond parallel map and per-request server tasks | v1 runs `par_map` and server handler tasks on worker threads in release binaries (G11, G13, ADR-044, ADR-051); programs spawn no threads or tasks of their own, and RC counts are non-atomic. | Planned (ADR-040): program-spawned share-nothing tasks with channels |
| Shared memory between threads | None; any future concurrency is share-nothing (ADR-027). | Excluded (ADR-040) |
| Loop constructs | Iteration is recursion with `decreases` measures (ADR-024). | Revisit on evidence (ADR-038, H1 measurement G9 and corpus P1) |
| Cycle collection | Not needed: acyclicity is a language invariant (ADR-023). | Excluded (ADR-039) |
| Mutable references | None in the language. In-place mutation happens only through FBIP reuse. | Excluded (ADR-039) |
| FFI from `.opq` | `.opq` cannot call foreign code. Foreign code runs only inside host providers. | Excluded (ADR-041) |
| User-defined effect handlers in `.opq` | Handlers are bound at the program entry point in `.cprime`. | Revisit on evidence (ADR-034, a P1 corpus program that needs them) |
| Side-channel resistance | Timing and cache behaviour are not constrained. | Revisit on evidence (ADR-034, a P1 corpus program that needs it) |
| Built-in network and filesystem providers | None ship with C′; a project may supply its own through the provider SDK. | Substitute (ADR-034): user-supplied providers |
| Dynamically loaded providers | Providers are statically linked into a project-specific runtime. | Excluded (ADR-041) |
| Capability tokens as runtime values | A capability is a compile-time set of effect operations only. | Revisit on evidence (ADR-034, a P1 corpus program that needs them) |
| Guarantees about behaviour the contract does not state | Properties absent from `ensures` are unverified, by definition. | Excluded (ADR-034) |

## Glossary

| Term | Definition | Example |
|---|---|---|
| C′ | The language and toolchain this HLD describes; pronounced "C-prime", ASCII `C'`, identifier stem `cprime`. Named "Pact" before ADR-049. | `cprimec build --release` |
| Contract | A specification a person approves: types, capabilities, `requires` and `ensures`, written in a `.cprime` file (ADR-046, ADR-050). Called a "pact" before ADR-050. | `auth.cprime` |
| Statement terminator | The end-of-statement marker the `.cprime` lexer inserts at a newline after an identifier, literal, `)`, `]`, `}`, `proved`, `tested` or `variant`; a line ending in an operator or `,` continues (ADR-050). | `requires user_id > 0` ends at its newline |
| C′ | The language, and the project as a whole. | — |
| `cprimec` | Thin command-line front end: builds the project-specific runtime and release binaries. | `cprimec` release build of a project |
| `cprimed` | Long-running compiler daemon. Holds the in-memory semantic graph and serves the RPC/MCP interface. | One `cprimed` per project |
| `.cprime` file | Human-approved surface, written by a person or drafted by an agent from an agreed plan (ADR-046): module, types, capabilities, specs, entry point, bindings. | `auth.cprime` |
| `.opq` file | Agent-authored surface: one S-expression implementation of one spec. The extension is short for *opaque*: people are not expected to read these files line by line (ADR-001), only the contract they implement. Kept by the user on 2025-12-20, after RFC 05 renamed contract files to `.cprime`. | `auth/generate_token.opq` |
| Module | Named unit declared by `module <path>` at the top of a `.cprime` file. | `module Auth` |
| Spec | A named function contract: signature, `uses`, `assurance`, `unconstrained`, `requires`, `ensures`. | `spec generate_token(...)` |
| Implementation (`impl`) | An `.opq` definition that names a spec and binds its parameters positionally. | `(impl Auth::generate_token [uid now secret] ...)` |
| Capability | A named set of effect operations. | `capability CryptoCap { ... }` |
| Effect operation | One operation of a capability, optionally marked `deterministic`: same result for the same arguments, no observable effect, and returns without waiting on anything outside the call (ADR-054). | `CryptoCap::hmac_sha256` |
| Effect set | The set of capabilities a function may use. Checked as a subset of the spec's `uses` list. | `{CryptoCap}` |
| `Diverge` | Built-in capability that exempts a function from the termination check. | `uses Diverge` |
| Host provider | Rust code, compiled into the runtime, that implements one capability. The only place foreign code runs. | `host::Crypto` |
| Binding | A `.cprime` declaration associating a capability with a provider at the entry point. | `bind CryptoCap = host::Crypto` |
| Server | A release program declared by a `server` item; it runs until stopped and serves requests (ADR-051). | `server Api { handler Api::handle; transport acme::HttpTransport }` |
| Handler | The spec a server runs once per request, in its own task. | `Api::handle` |
| Transport | A provider implementing the transport interface, which delivers requests and sends responses. | `acme::HttpTransport` |
| Task | A share-nothing unit of execution with its own task heap. In v1 only the runtime spawns tasks, one per server request (ADR-051). | — |
| Task heap | The context-private memory a task runs in; objects are freed by reference counting, and the whole heap is freed if the task ends abnormally. | — |
| Entry point | The `.cprime` `entry` declaration a program starts from. | `entry main(...) ... { uses ... }` |
| Provider SDK | The `cprime-provider` crate every provider implements. | — |
| Project manifest | `cprime.toml` at the project root: the provider crates a project uses and its configuration. Its presence marks the project root. | `[[providers]] crate = "cprime-zstd-provider"` |
| Project-specific runtime | Cargo build that statically links a project's providers; used by both `cprimed` and release builds. | — |
| Provider inventory | Per-release-build report of every linked provider crate and its exact version from `Cargo.lock`. | — |
| `DefId` | Blake3 hash of a definition's canonical serialisation (references by `DefId`, locals by de Bruijn index). | 32-byte hash |
| Namespace table | Mapping from qualified names to `DefId`s. | `Auth::generate_token` → `DefId` |
| Context | The definitions transitively reachable from a spec, returned by `get_context`. | Types, capability signatures, prelude functions |
| Salsa database | Incremental-computation store inside `cprimed`. A cache, not the source of truth. | — |
| C′-IR | SSA intermediate representation with block arguments, terminators and traps. | `cprime-core/src/ir.rs` |
| Perceus | Compile-time reference-counting insertion (`RcDup`/`RcDrop`). | — |
| FBIP reuse | Functional-but-in-place: reusing a dropped value's memory for a same-layout construction when its count is 1. | `DropReuse` → `MakeStruct { reuse }` |
| Trap | Program abort on overflow, division by zero, out-of-bounds index or unreachable code. | `TrapKind::Overflow` |
| Proof obligation | One verification goal: an `ensures` clause, a potential trap site, a callee's `requires` at a call site, or a `decreases` measure on a call edge. | — |
| Counterexample | Inputs satisfying `requires` that violate an obligation, found by Z3 or fuzzing. | `now: 100` → `expires_at: 1100` |
| Evidence level | The strength of verification achieved: `proved`, `tested`, `none`. | `proved` |
| Assurance policy | Per-spec minimum evidence level: `proved` (default) or `tested`. | `assurance proved` |
| Diagnostic | A structured finding with a kind, tier, severity and node id. | `ContractViolation` |
| Tier (T0–T3) | Latency-budgeted stage of checking: front end, IR/JIT, SMT, fuzzing. | T2 = SMT |
| Unconstrained waiver | Spec clause naming result fields the human knowingly leaves unconstrained. | `unconstrained [nonce]` |
| Trusted computing base (TCB) | Components whose bugs can break the language's guarantees: `cprimec`/`cprimed`, Z3, Cranelift, LLVM, host providers. | — |
| H1 | Unmeasured hypothesis that `.opq` uses ≥ 40% fewer tokens than idiomatic Rust at equal acceptance rate. | — |
| IR verifier | Well-formedness check of C′-IR run after lowering and after every IR transformation. | Rejects a block argument count that does not match its target |
| Assumed provider operations | The `deterministic` provider operations a `proved` or `tested` result relies on being correct. | `CryptoCap::hmac_sha256` |
| Lexicographic measure | A tuple of `U64` expressions that must decrease, compared left to right, on every recursive call in a component. | `(decreases [n m])` — syntax fixed in `/spec` |
| Acceptance rules | The mapping from assurance policy, solver results and fuzzing results to accept/reject and an evidence level. | ADR-025 |
| Acceptance header | First line of an accepted `.opq`, recording format version, verification context hash, evidence level and assumed provider operations. | `;; cprime-accepted v1 context=9f…e1 evidence=proved assumed=Auth::CryptoCap::hmac_sha256` |
| Verification context hash | Blake3 hash over the sorted `DefId`s of the spec and every definition its verification depends on: its types, the capabilities in its `uses`, and the specs its implementation calls. | 32 bytes, written as 64 hex characters |
| Stale implementation | An accepted implementation whose recorded verification context hash differs from the current one. | `auth/generate_token.opq` after `generate_token`'s `ensures` is edited |
| Call-precondition obligation | Proof obligation that a call site satisfies the callee spec's `requires`. | `requires of Math::div_exact: d > 0` |
| Definedness rule | A contract clause holds only if it is defined and true. Division or remainder by zero, `index` out of range, `slice` out of range and access to a field of a variant the value does not have are undefined. Short-circuit operators evaluate their right operand only when needed. | `result is Ok ==> result.Ok.value > 0` is defined for every result |
| Variant test | Contract expression `e is V`, true when sum value `e` is variant `V`. | `result is Err` |
| Fuel | Count of function entries an interpreted or JIT-compiled call may perform before it aborts with the CPU-time limit. | `fuzz.call_fuel = 10000000` |
| Call-depth limit | Maximum nesting of C′ calls before a call traps `StackLimit`. | `limits.max_call_depth = 100000` |
| Launcher | `cprimec daemon`: checks the runtime fingerprint, rebuilds the runtime if stale, then replaces itself with the project's `cprimed`. | Agent host config: `cprimec daemon --project .` |
| Runtime fingerprint | Blake3 hash over the `cprimec` version string, the `[[providers]]` entries of `cprime.toml`, `runtime.cprime_source`, the `Cargo.toml` and `src/` tree of every path provider, and the `Cargo.lock` of the generated runtime package. | `.cprime/runtime.fingerprint` |
| Value JSON encoding | The JSON form of C′ values at every JSON boundary (SA-17). | `{"Ok": {"value": 3}}` |
| Intrinsic | A prelude function implemented by the runtime, not by a spec: `len`, `index`, `++`, `u64_be`, `push`, `slice`, `to_bytes`. | `IntrinsicOp::U64Be` |
| `Unit` | Type with one value; the result of operations performed only for their effect. | `host::Log::info` returns `Unit` |
| Expressiveness parity | Every program a Rust 2024 developer writes has a C′ formulation that keeps the four guarantees and meets G10; the construct may differ (ADR-034). | A doubly-linked list written as an arena |
| Expressiveness corpus | The program set P1 is measured on: the 50 H1 tasks plus 20 selected Rust programs. | — |
| Post-v1 direction | One of Planned, Substitute, Revisit on evidence or Excluded, recorded per non-goal (ADR-034). | `Excluded (ADR-039)` |
| Future-reserved word | A word with no v1 meaning that v1 rejects as a name, so a later version can give it one (ADR-043). | `loop` in `.opq` |
| Reserved type name | A type name v1 forbids declaring because a later built-in type will claim it (ADR-043). | `Str` |
| Spec-typed parameter | Post-v1: a parameter whose type names a spec; calls through it are verified against that spec's contract (ADR-037). | `f: Hasher` |
| Interface | Post-v1: a `.cprime` declaration of operation signatures over a type parameter, with laws as contracts (ADR-036). | `interface Ord<T>` |
| Task | Post-v1: a unit of concurrency with its own heap that communicates only by messages (ADR-040). | — |
| Parallel map | The `.opq` form `(par_map S xs)`: applies spec `S` to every element of `xs`; parallel in release binaries, sequential everywhere else (ADR-044). | `(par_map Auth::verify_one tokens)` |
| Worker | A thread a release binary uses for one contiguous chunk of a parallel map. | — |
| Worker context | The `RtContext` of a worker (`mode` 2): its own heap, depth counted from the caller, memory counted against the shared limit. | — |
| Thread-safe provider | A provider whose `ProviderDecl` sets `thread_safe: true`, so `par_map` may call its operations from several workers (ADR-045). | `host::Crypto` |
| Witness trace | The executed steps a counterexample's failing clause or trap depends on, replayed through the interpreter and attached to the diagnostic (ADR-047). | `exp = 1100` at `(+ now 1000)` |
| Replay status | Whether a witness trace reproduced the reported outcome: `reproduced`, `trap_first` (a trap was reached first) or `not_reproduced` (a TCB defect). | `reproduced` |
| Solver-chosen value | A value the replay took from the Z3 model rather than computing: a called spec's result, a provider operation's result or a parallel-map result. | `sig` from `hmac_sha256` |
| Trap site table | The table generated into a release binary that maps each trap site to its spec, file, line and column (ADR-048). | `site 3 → Auth::generate_token auth/generate_token.opq:3:13` |
| Call chain | The spec calls active when a release binary failed, outermost first; reported only by builds with `release.call_chain = true` (ADR-048). | `Main::run → Auth::generate_token` |
| Vacuous contract | A contract whose `requires` no input satisfies, or whose `requires` and `ensures` no input–result pair satisfies; rejected with `VacuousContract` (ADR-052). | `requires x > 5 && x < 3` |
| Effective evidence | The weakest evidence level across an accepted implementation and every implementation it transitively calls, computed when reported (ADR-053). | `proved`, effective `tested` |
| Known-answer vector | A fixed argument tuple and expected result, taken from a standard, against which `cprimec runtime build` checks a `deterministic` operation (ADR-054). | RFC 4231 test case 1 |
| Call outcome | How a call ends: returns, traps, exceeds a resource limit, provider fails, or diverges (ADR-024). | trap `Overflow` → exit 3 |
| Concrete counterexample | A counterexample whose violation recurs with real `deterministic` providers and accepted callee implementations (ADR-055). | wrong expiry in `generate_token` |
| Abstract counterexample | A counterexample valid in the verifier's model that did not recur, or could not be run, concretely; carries a reason (ADR-055). | an all-zero `hmac_sha256` result |
