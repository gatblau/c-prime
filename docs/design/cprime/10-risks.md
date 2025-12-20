# C′ — Risks, Assumptions, Open Questions, Alternatives, Dependencies, Rollout

> **Mode:** draft
> **Generated from:** inline context — `docs/cprime-concept.md` (Draft, revision 2)
> **Revision:** 0.12.0
> **Last updated:** 2025-12-20

## Assumptions

| ID | Area | Assumption | Rationale |
|---|---|---|---|
| A-1 | Security | `cprimed` accepts connections from the local machine only and performs no authentication in v1; the trust boundary is the local user account. | The concept names no auth; a local developer daemon is the stated use. |
| A-2 | Tenancy | One `cprimed` instance serves exactly one project. | Start-up loads "the" `.cprime`/`.opq` files and one project-specific runtime. |
| A-3 | Concurrency | Any number of `cprimed` instances may run on one project, one per agent session; each serves one client over stdio and runs its requests one at a time. Writes of accepted implementations are serialised by a project write lock held only while writing, and a write is refused with a conflict when the target `.opq`, or a `.opq` the verification read, changed on disk after the request began. | Concurrent agent sessions on one project are common (user decision, 2025-12-20). Disk is authoritative (ADR-003) and re-read before every request (ADR-030), and verification is modular (ADR-033), so sessions can collide only on writes to the same files. |
| A-4 | Logging | `cprimed`/`cprimec` log structured lines to stderr; submitted source and counterexamples are omitted at default verbosity. | Not stated in the concept; a conservative default. |
| A-5 | Deployment | `cprimed` and `cprimec` run on a developer workstation or CI host; there is no hosted service. | Concept describes a local daemon and CLI only. |
| A-6 | Scale | Latency targets apply per implementation (≤ 200 IR instructions), not per project; no project-size limit is set in v1. | The concept bounds only the implementation size. |
| A-7 | Persistence | The Salsa database is not persisted across restarts; it is rebuilt from disk on every start-up. | "The daemon is a cache"; start-up loads files. |
| A-8 | Privacy | C′ sends no telemetry off the machine. | No external service is named. |
| A-9 | Observability | `timings_us` in replies is the only metrics surface in v1; witness traces and release trap locations are diagnostics, not metrics. | Concept example response. |
| A-10 | Configuration | Configuration is read from a file and command-line flags at process start; no reload while running. | Conservative default; keys are fixed in `/spec`. |
| A-11 | Toolchain | A Rust toolchain with Cargo is installed wherever `cprimec` builds the project-specific runtime. | The runtime is "built with Cargo". |
| A-12 | Version control | C′ never invokes version control; the user commits accepted `.opq` files. | Concept: "version control therefore records" acceptances. |
| A-13 | Crate graph | `cprime-provider` does not depend on other workspace crates, so third-party providers pull in only the SDK. | Keeps provider builds small; not stated in the concept. |
| A-14 | Persistence | If the atomic write fails, `submit_impl` reports a daemon failure and does not report acceptance; an interrupted submission writes nothing. | Follows from "write ... before replying". |
| A-15 | Build | A failed runtime build or an invalid `bind` stops the release build; no partial binary is emitted. | Conservative default. |
| A-16 | Evidence | The evidence level and assumed provider operations of an accepted implementation are stored in its acceptance header; solver obligations are not stored (OQ-26). Effective evidence across callees is computed when reported, never stored (ADR-053). | ADR-031, ADR-053. |
| A-17 | Errors | An IR verifier failure is a compiler bug and is reported through the JSON-RPC `error` object as a daemon failure, not as a rejection. | The agent cannot fix a compiler bug, so it must not look like a rejection of its code (ADR-010). |

## Open Questions

All open questions are **non-blocking**. As of revision 0.3.0 every question except OQ-20 has a resolution; rows are kept for traceability. Ids continue from the concept's OQ-1 to OQ-10, which are now ADR-012 to ADR-021.

| ID | Question | Plain-English | Options | Impact | Blocking? | Resolution |
|---|---|---|---|---|---|---|
| OQ-11 | Which local transport carries JSON-RPC and MCP? | How does the AI tool talk to the compiler running in the background? Standard input/output is simplest but ties one tool to one daemon; a local socket lets several tools share one daemon but needs access control. | stdio; Unix domain socket; loopback TCP | `cprime-daemon` server shape; A-1 | No | ADR-029 (stdio) |
| OQ-12 | How does `cprimed` learn that a human edited a `.cprime` file? | When a person changes a contract, how does the running compiler notice? Watching files is automatic but adds moving parts; an explicit reload is simpler but easy to forget. | File watching; explicit reload call; re-read on every request | Salsa input updates; sequences | No | ADR-030 (per-request sync) |
| OQ-13 | What happens to an accepted `.opq` when its spec changes? | If a contract changes after code was accepted, does that code get re-checked? Re-checking keeps everything trustworthy but costs time; not re-checking is fast but can leave stale code marked as fine. | Re-verify on change and report; mark stale only; ignore until resubmitted | Implementation lifecycle; trust argument | No | ADR-031 (stale, not re-verified) |
| OQ-14 | How are on-disk formats and `DefId`s versioned across compiler releases? | When the compiler is upgraded, do old files and hashes still work? A version marker allows clean upgrades but adds a field everywhere; no marker is simpler but upgrades may silently change hashes. | Language version marker; hash-scheme version in `DefId`; none in v1 | Incremental cache; file formats | No | Hash domain tag `cprime.def.v1`; mutually recursive definitions hashed as a group; header format `v1` (spec SA-28, SA-37) |
| OQ-15 | What is the `cprimec` command set, and which crates does it link? | What commands does the command-line tool offer? Defining them now constrains the specs; leaving them to the spec phase is faster but less reviewable. | Define in `/spec` from the roadmap; define here | CLI interface; architecture | No | `cprime-cli` crate and command set (`02-architecture.md`, `08-interfaces.md`) |
| OQ-16 | Which Z3 and LLVM versions, and are they bundled or taken from the system? | Do we ship the solver and the back-end compiler ourselves? Bundling makes installs reliable but bigger; using the system copy is lighter but versions vary per machine. | Bundle; system install; build from source | Install story; reproducibility | No | ADR-028 |
| OQ-17 | Which operating systems and processor types are supported in v1? | Which machines must C′ run on? Fewer platforms means faster delivery; more platforms means more testing for the JIT and release builds. | Linux x86-64 only; plus macOS arm64; plus Windows | JIT, AOT, CI matrix | No | ADR-028 |
| OQ-18 | How does a release binary receive its entry-point arguments and report its result? | How is a finished program started and how does it hand back its answer? Fixing this early shapes the runtime; deferring it keeps v1 smaller. | Command-line arguments; standard input; embedding API | Runtime, entry point | No | JSON on stdin/stdout, exit codes (`08-interfaces.md`) |
| OQ-19 | Which crate owns the `UnconstrainedOutput` check? (The sum-type rules are decided in ADR-026.) | Which part of the compiler runs the "every output must be described" check? | `cprime-types`; `cprime-effects`; `cprime-verify` | Component boundary; ADR-017, ADR-026 | No | `cprime-types` (`02-architecture.md`) |
| OQ-20 | Who are the first users, and what evidence motivates the project? | Who is this for first, and what problem have they actually hit? Naming them sharpens priorities; leaving it open keeps the scope general. | Name pilot users/projects; keep as research project | Motivation; roadmap priority | No | Open |
| OQ-21 | What happens when fuzzing cannot find inputs that satisfy `requires`? | If the random tester cannot find valid inputs, does the code pass or fail? Failing is safe but may block valid code; passing is unsafe. | Reject with a dedicated diagnostic; report `none`; smarter generation | Fuzzing harness; diagnostics | No | `VerificationInconclusive` at T3 with `reason: fuzz_budget_incomplete` |
| OQ-22 | What does `cprimed` do if the project-specific runtime is missing or out of date? | If the compiled helper code is stale, does the daemon rebuild it itself or refuse to start? Rebuilding is convenient but slow on start-up; refusing is predictable but needs a manual step. | Auto-rebuild; refuse with instructions; run with built-ins only | Start-up sequence | No | ADR-032 (launcher rebuilds) |
| OQ-23 | What licence review applies to the compiler's dependencies and to code linked into release binaries? | Are there licence obligations when we ship binaries containing other people's code? Reviewing now avoids surprises; deferring is quicker. | Review in `/spec`; defer to Phase 5 | Regulatory table; release | No | Deferred to Phase 5; inventory records licence expressions |
| OQ-24 | What are the project manifest's name, location and format? | What file lists the add-on libraries a project uses? (The concept already defers this to the spec phase.) | Decide in `/spec` | Interfaces | No | `cprime.toml` (`08-interfaces.md`) |
| OQ-25 | What is the full concurrency policy for multiple clients? | Can two AI agents work on the same project at once? Allowing it speeds up work but risks conflicting writes; disallowing it is simpler. | One client only; serialise per spec (A-3); serialise all submissions | `cprime-daemon`; A-3 | No | A-3 as amended (several daemons per project, write lock, conflict check) |
| OQ-26 | Should the solver's proof obligations (SMT-LIB) be saved beside each accepted `.opq`? | Should we keep a copy of the exact maths problem the solver checked, so a "proved" result can be re-checked later or on a newer solver? Keeping it makes results reproducible but adds files to every project; not keeping it is tidier but "proved" depends on one solver version. | Persist beside the `.opq`; persist in a cache directory outside version control; do not persist | ADR-003, A-16, data lifecycle | No | Not persisted in v1 |
| OQ-27 | Can providers ship their own contracts, fuzz-checked when they are bound? | Should add-on libraries come with their own promises that the compiler tests? It would make "proved" mean more, but it is a new feature. Recommended answer: not in v1. | v1 non-goal (recommended); include in v1 | Provider SDK, verifier | No | v1 non-goal |
| OQ-28 | Can a client cancel a submission that is still running? | Can the AI tool stop a long check it no longer needs? Allowing it saves time but adds protocol and daemon work; not allowing it is simpler. | JSON-RPC cancellation; no cancellation in v1 | `cprime-daemon`, interfaces | No | No cancellation in v1 |
| OQ-29 | What are the module, visibility and import rules for `.cprime`? | How does one contract file use types from another, and can some definitions be private? | Explicit imports; fully qualified names only; visibility markers | Grammar, namespace table | No | No imports, no visibility; resolution order (`08-interfaces.md`) |
| OQ-30 | Does `.opq` have first-class functions? | Can agent-written code pass functions around as values? It adds expressive power, but complicates termination checks and the no-cycles rule. | No first-class functions in v1; non-capturing functions only; closures | `Type::Fn` use, ADR-023, ADR-024 | No | No first-class functions in v1 |

### Must be settled in `/spec` before a phase starts

These are low-level details, not HLD questions; they are listed here so `/spec` and `/breakdown` treat them as gates.

| Item | Needed before | Source |
|---|---|---|
| Normative grammar of both surfaces, including `decreases` tuples and variant tests | Phase 1 | ADR-023, ADR-024, ADR-026 |
| Parser recovery anchors (top-level forms, closing parentheses in `.opq`) | Phase 1 | Review finding i |
| Module, visibility and import rules | Phase 1 | OQ-29 |
| RC on block arguments, `Switch` payload bindings and trap edges; `DropReuse` token type; location of the count-is-1 test | Phase 2 | ADR-022 |
| IR additions: a unit type, constant vectors and structs; IR verifier rules | Phase 2 | Review finding g |
| Evaluation order, checked arithmetic and trap behaviour, identical in interpreter, JIT and AOT | Phase 2 | Review finding h |
| Provider invocation ABI | Phase 3 | Review §4 |
| Full acceptance table; obligation parallelism and cut-off order; SMT theories and encoding; operational meaning of contract expressions | Phase 4 | ADR-025 |

## Risks

| Risk | Mitigation |
|---|---|
| Contract incompleteness: accepted code is correct only against what `ensures` states; agents may game weak contracts (e.g. returning `Err` for every input). | `UnconstrainedOutput` with explicit waivers; sum-type completeness rule (ADR-026); document that unstated behaviour is unverified. Vacuous and inconsistent contracts are rejected (ADR-052). |
| `proved` is read as "the program is correct", when it holds only relative to provider models. | Replies list the assumed provider operations and the effective evidence across callees (ADR-053); counterexamples state whether they are concrete (ADR-055); provider contracts are OQ-27. |
| `proved` is not reproducible across Z3 versions. | Pin the Z3 version (ADR-028); persisting obligations is OQ-26. |
| Z3 returns `unknown` often under `assurance proved`, stalling agents. | Measure the inconclusive rate in Phase 4; parallel obligations and configurable timeouts (ADR-025); `assurance tested` as an explicit opt-in. |
| H1 is refuted (no token saving or lower acceptance). | Phase 3 reports the result either way; the project proceeds regardless (see Motivation); revisit the `.opq` surface before Phase 4 if refuted. |
| The Go-like contract syntax (ADR-050) is not easier to read or write in practice; the token counts (196 → 160 on the concept specs) are a proxy, not a measurement. | Hypothesis H2, measured in Phase 3 alongside H1: 8–10 developers new to C′ each review 10 contracts in both syntaxes, in a counterbalanced order, each contract with one planted defect (a wrong bound, or a result field missing from `ensures`); record defects found and time taken. They then write contracts from plain-English descriptions; record parse errors on the first attempt. Pass condition: the new syntax is not worse on either measure; otherwise the result is reported and ADR-050 revisited. |
| A transport or a provider declared thread-safe is not safe under concurrent calls, corrupting state or responses (ADR-051). | The claim is part of the trusted computing base (ADR-045); the sample transport and the built-ins carry a concurrency stress test; the release provider inventory lists every transport. |
| Returning to a task's recovery point after a trap skips work the trap entry's caller expected to run (ADR-051). | C′ frames hold only task-heap memory and the trap entry holds no values with destructors; a test traps at every site kind in a handler 10,000 times and checks that live memory returns to its baseline and every later request succeeds. |
| RC insertion bugs at calls, branches or reuse sites cause leaks or use-after-free. | Calling convention fixed by ADR-022; IR verifier after every transformation; leak harness in Phase 2. |
| Release builds miss the 2× Rust target because all heap values are reference counted. | Phase 5 benchmarks; region inference is the recorded follow-up. |
| False `deterministic` claims by providers make proofs unsound. | Double-call probe (`NonDeterministicProvider`); provider inventory makes trusted code visible; document provider author responsibility; built-in operations are checked against known-answer vectors at runtime build (ADR-054); the effect-free obligation is documented in the SDK. |
| Bugs in the TCB (`cprimec`/`cprimed`, Z3, Cranelift, LLVM, providers) break guarantees. | IR interpreter as oracle; JIT-vs-interpreter differential tests; pinned versions (ADR-028). |
| JIT and AOT back ends diverge in behaviour. | Differential tests across interpreter, JIT and AOT on the corpus, including trap locations (`cprime-codegen` Part 6). |
| LLVM/`inkwell` version churn breaks builds. | Pin versions (ADR-028). |
| Spec edits leave previously accepted implementations unverified. | Stale marking; release builds refuse stale code (ADR-031). |
| Rejection sampling starves under tight `requires`, making `assurance tested` unusable for some specs. | Reported as `VerificationInconclusive` at T3 with a `reason`; measure in Phase 5. |
| Latency budget missed on larger projects as the Salsa graph grows. | Targets are per implementation (A-6); measure with growing corpora in Phase 3. |
| Side channels and resource exhaustion in release programs. | Out of scope (side channels); configured CPU-time, memory and call-depth limits abort runaway programs. |
| The first `cprimec daemon` launch, and every launch after a provider change, runs a Cargo build that needs registry access or a local C′ source checkout. | `runtime.cprime_source` points at a local checkout for offline use; the build output is written to stderr; the fingerprint is written only after success. |
| Boxing every struct and sum value (including scalar-only ones) costs allocations the concept did not foresee. | Measured by the Phase 5 benchmarks; unboxing scalar-only values is a follow-up behind the same calling convention. |
| The per-request source scan grows with the project's file count. | Reported separately as `timings_us.sync`; measured with growing corpora in Phase 3. |
| Native stack frame size of JIT and AOT code bounds safe recursion depth. | Call-depth limit with `StackLimit` trap; frames ≤ 1 KiB checked by a corpus test; entry thread sized from the limit. |
| A provider built with `panic = "abort"` turns a provider panic into a whole-program abort; a provider that keeps argument pointers breaks memory safety. | Generated runtime packages force `panic = "unwind"`; the SDK documents the no-retention rule; providers are in the TCB and listed in the inventory. |
| Generated runtime and release builds depend on registry access. | Same mitigation as the first-launch build risk (`runtime.cprime_source`). |
| Several `cprimed` instances on one project each keep a full Salsa cache and their own solver instances. | Resident memory per daemon is measured in Phase 3 (expected below 512 MiB for a 500-file project); the number of sessions is the user's choice. |
| Two sessions repeatedly resubmit the same spec and keep conflicting. | The conflict reply names the changed file; the agent re-reads it with `get_context` before resubmitting; each refused submission costs one verification. |
| An edit that leaves a file's size and modification time unchanged is not seen by the per-request scan (ADR-030). | Modern filesystems record modification time to the nanosecond, so this needs two same-size writes within one tick; a content-hash fallback is a follow-up candidate if it is observed. |
| Expressiveness parity is unmeasured: some Rust programs may have no acceptable C′ formulation under the exclusions of ADR-039 to ADR-041. | P1 measures it on the expressiveness corpus and reports the result; an exclusion is reopened only by a superseding ADR. |
| Until tail calls ship (ADR-038), iteration by recursion traps `StackLimit` beyond `limits.max_call_depth` (default 100,000; maximum 10,000,000). | Raise the limit in `cprime.toml`; tail calls are planned post-v1 and relax the trap without changing any successful result. |
| Floating-point obligations may exceed the T2 budget, so floating-point code may reach only `tested` evidence (ADR-042). | `assurance tested` is the default for such specs; measure solver time before the feature ships. |
| `par_map` copies every element in and every result out; for cheap per-element work the copies can cost more than the parallelism saves. | Sequential path below two elements or one worker; the G11 benchmarks report the speed-up, including where it is below 1. |
| A provider that declares `thread_safe` falsely can corrupt state or results inside `par_map` (ADR-045). | The flag is in the provider's source and joins the TCB like `deterministic`; Rust's `Sync` rule covers safe code; built-in providers opt in only for `host::Crypto`. |
| When a resource limit is hit inside `par_map`, which element hit it, and therefore which trap is reported if an ordinary trap also occurs, may differ between runs. | Stated in ADR-044 and G11; ordinary traps alone are deterministic. |
| Each worker thread reserves a stack of `max_call_depth × 1 KiB + 1 MiB` (about 99 MiB at the default limit), so 256 workers reserve about 25 GiB of address space. | Address space, not memory; `limits.workers` and `limits.max_call_depth` bound it. |
| An agent-drafted contract is approved without a real review, or implemented before approval, so accepted code faithfully meets a contract that does not match the plan (ADR-046). | C′ cannot detect this; the spec-driven process requires a person to approve each contract and records the approval in version control. `UnconstrainedOutput` and the completeness rule (ADR-026) point the reviewer at gaps. |
| Solver-chosen values that no real provider returns (an all-zero HMAC) confuse agents reading a witness trace. | Every such step is marked `solver`; RFC 03 criterion 17 measures the effect on the H1 corpus; the counterexample is marked `abstract` when the concrete replay does not reproduce it (ADR-055). |
| A `not_reproduced` trace is a false alarm caused by the replay itself rather than the encoder. | Logged at `error` with the obligation; the corpus measurement counts them; each one is investigated as a TCB defect. |
| The `perform` site store and inline `slice` check cost run time in default release builds. | Measured by the Phase 5 benchmarks against G10 (RFC 03 criterion 16). |
| Opt-in call-chain builds use 4 bytes × `limits.max_call_depth` per context (400 KB at the default) and one store per call. | Opt-in only; overhead reported separately in the Phase 5 benchmarks. |
| Information flow: a permitted operation (for example `Log::info`) may receive a secret; capabilities give no confidentiality guarantee. | v1 limit, stated in the security posture (`07-cross-cutting.md`); information-flow control is out of scope (RFC 07). |
| Provider authority: a provider's native code may do more than its declared operations, and the inventory cannot show otherwise. | v1 limit; providers are in the TCB, listed in the inventory, and never described as verified. |
| A third-party `deterministic` operation without known-answer vectors computes the wrong function, so proofs that assume it do not hold for the program. | The inventory shows `"vectors": 0`; the runtime build logs a warning; concrete replay surfaces mismatches as `abstract` counterexamples (ADR-055). |
| An abandoned provider thread keeps running and holds memory until its provider returns (ADR-054). | At most eight per `cprimed`; stubs stay installed, so it performs no real I/O; restarting `cprimed` reclaims everything. |
| Concrete replay runs callee and provider code, so rejections take longer (ADR-055). | Bounded by `fuzz.call_fuel`, `fuzz.call_memory_bytes` and `verify.provider_wait_ms`; reported in `timings_us.witness`. |

## Alternatives considered

- **Agents write idiomatic Rust and humans keep reviewing it** — the status quo and the H1 baseline. Rejected: keeps line-by-line human review as the gate and gives no machine-checked contract.
- **Adopt an existing verification-oriented language instead of a new one** *(drafted — the concept does not discuss this)*. Rejected for v1 on the argument below. The argument is a draft for the user to confirm or correct.

### Why not an existing verification language?

**Question.** Why not give an agent Dafny, Verus, F\*, Lean, SPARK or Why3, rather than build C′?

**Status.** Draft. Statements about C′ cite this repository. Statements about the other systems are **external claims** (EV-1 to EV-6, with results in the table at the end). They were checked on 28 October 2026 by web search and page fetches, mostly from documentation excerpts and abstracts, not from a full read of each manual or paper. Re-check anything marked *Open* or *Partly* against the primary source before this argument is quoted outside the repository.

**Answer.** An agent can already be put in front of any of these systems, and published work does that (DafnyBench, Clover, AlphaVerus, Marmaragan, ExVerus). The open question is what the *author-facing language* should contain when the author is a model. The existing systems give the author a proof vocabulary (loop invariants, ghost code, lemmas, `assume`) because a person is expected to use it. C′ removes that vocabulary from the agent's surface and moves the human's attention to a contract with no control flow. The case for a new language therefore rests on three design differences that need ownership of the language (A to C below). Two further points that looked like differences on first reading (D and E) are weaker, and this section says so. None of the benefits is measured yet.

#### The claims, graded by evidence

| # | Claim | Evidence in this repository | Evidence outside it (EV-n) | Grade |
|---|---|---|---|---|
| A | The agent-facing surface carries no proof vocabulary | `.opq` has eleven expression forms (literal, name, `let`, `if`, `match`, `perform`, field access, `vec`, `par_map`, operator application, path application; `docs/spec/components/cprime-syntax.md`, `.opq` grammar). It has no loop, invariant, assert, assume or lemma form. `loop` and `invariant` are reserved against future use. Repetition is recursion with a `decreases` measure (ADR-024). | Annotation writing is where reported agent effort goes: DafnyBench tests LLMs on supplying verification hints; Marmaragan generated correct SPARK annotations for 50.7% of its cases with GPT-4o; ExVerus targets invariant generation for Verus (EV-6) | **Premise supported; remedy untested.** Whether removing the vocabulary lowers cost is H1 |
| B | The author cannot reach an escape hatch from `.opq` | No `assume`, `assert`, `admit` or axiom form in the grammar above | Each system has one: Dafny `assume` and `{:axiom}`; Verus `assume`, `admit`, `external_body`; F\* `assume` and `admit`; SPARK `pragma Assume`; Lean `sorry`. Dafny and Verus have a mode that bans them (EV-5) | **Supported, and narrower than it sounds.** The others can ban the hatch; C′ cannot express it. C′ has its own trust points (below) |
| C | The agent never meets a borrow checker | No borrow annotations in `.opq`; Perceus reference counting instead (`README.md`) | Verus relies on Rust's borrow checker, so a Verus author meets it and the proof burden together (EV-3) | **Supported** |
| D | Authority is a granted, reviewed capability set with no ambient default | `uses` clause in `.cprime`; no FFI from `.opq`; provider inventory per build (`README.md`) | F\* tracks effects (PURE, DIV, STATE, EXN). Nothing found shows that any of the six restricts a program to a granted capability set by default (EV-2) | **Unconfirmed.** Possibly not new against F\*. The defensible claim is the combination: a per-function grant a person approves, enforced with no ambient default |
| E | The counterexample loop and agent tool interface | Structured diagnostics with counterexamples; MCP tools; latency tiers (`README.md`) | Community MCP servers exist for Dafny and Lean; ExVerus uses counterexamples to repair Verus proofs (EV-4, EV-6) | **Not a differentiator.** Keep as a design requirement, not as a reason for a new language |
| F | Release performance within 2× of hand-written Rust | Target G10; unmeasured | None checked | **Unmeasured** |

#### What C′ also trusts

Claim B is not "C′ has no trust points". The trust points sit where the agent cannot write them:

- `assurance tested` in a `.cprime` spec accepts fuzzing in place of proof, and the evidence level is recorded (ADR-025). It is a contract clause, so a person owns it.
- An `unconstrained` waiver in a `.cprime` spec accepts an unstated result field (ADR-017). A person owns it.
- Providers and their `deterministic` claims are Rust code outside the verifier, listed in the release inventory (ADR-054).
- The compiler, Z3, Cranelift and LLVM are in the trusted computing base (`README.md`).

The comparison with the other systems is therefore "who can reach the hatch", not "whether one exists".

#### What C′ gives up for claim A

This section records a cost the earlier draft did not name. It is inferred from the grammar and ADR-033 and has not been checked against a worked example.

- A proof about a recursive spec uses the spec's own contract for the recursive call, because verification is modular (ADR-033). The `ensures` clause then plays the part of the inductive invariant.
- If the contract is true but not strong enough to be inductive, the proof fails. In Dafny or Verus the author adds an invariant or a lemma. In `.opq` there is no form for either, so the fix is to strengthen the contract or add a helper spec. Both are `.cprime` changes, and a person owns `.cprime` (ADR-046).
- The agent may therefore stall on a correct implementation, with the person as the only way out. Phase 4 should count how often a rejection needs a contract change, and the H1 protocol should record it.

#### Why a new language rather than a wrapper

A wrapper around an existing verifier delivers claim E and nothing else. Claims A to C need a different author-facing language, because they remove constructs the existing systems provide and expect the author to use. Removing them from an existing system gives a subset. The project would then own a dialect and its verifier without having designed the surface for the agent. If a subset proves enough, that is a result worth reporting (see below).

#### Where the existing systems are ahead

State this plainly in any external text.

- They are mature, widely used and far more expressive than C′ v1, which is first-order, has no closures, no mutable references and no loops (`cprime-why.md`, "The implementation (`.opq`) is functional, with tight limits").
- A person can guide a proof when automation stops. C′ offers `assurance tested`, which is weaker evidence (ADR-025), and the contract change described above.
- Published results exist: DafnyBench (arXiv:2406.08467) reported a best result of 68% at publication, and a later paper claims 96% (secondary source, unverified). C′ has none.
- The other systems share C′'s main limit: they guarantee only what the contract states.

#### What would change this answer

1. An agent using Dafny or Verus, with its escape hatches banned, matches `.opq` on first-submission acceptance and tokens per task on the H1 corpus. Claim A would then have no measured benefit.
2. A restricted profile of an existing system enforces "no ambient authority" for agent-written code. Claim D would then not need a new language.
3. Rejections that need a contract change are frequent in Phase 4. The cost above would then outweigh claim A.
4. H1 is refuted. The case then rests on verified acceptance and authority alone (`README.md` says it does not rest on H1), and claim A is reopened (see the H1 risk above).

#### Proposed, not decided: a third arm for the H1 study

Add an existing verifier as a target on the same 50 tasks with the same model (concept OQ-9 defines the corpus and the two current arms). Three design points follow from the evidence:

- **Ban the hatches in the baseline.** Run the Dafny arm with `/noCheating` and the Verus arm with `--no-cheating`, so that no arm wins by assuming the answer. Check first what the Verus flag does not catch: one third-party report says an `axiom` function still passes it (EV-5).
- **Give every arm the same contract.** Write each task's contract once and translate it to each language by hand, so that the contract's strength is not a variable.
- **Record more than tokens.** Per arm: tokens per accepted task, first-submission acceptance, rounds to acceptance, and for C′ the number of rejections that need a contract change.

This changes the H1 protocol, so it must be settled before the protocol is registered. Results from different benchmarks are not comparable, so the existing published figures cannot stand in for it.

#### If C′ fails

The design question stands. The result would then be a specification for a profile of an existing system, not a verdict on the idea.

#### External claims, and what was found

| ID | Claim | Result (28 October 2026) | Source |
|---|---|---|---|
| EV-1 | Dafny, Verus, F\*, SPARK and Why3 keep specification and implementation in one language, and the author writes invariants, ghost code or lemmas beside the implementation | **Confirmed in part.** Dafny: loop invariants, ghost constructs and lemmas; it infers only simple invariants. Verus: specification, proof and executable code interleaved under a mode system. SPARK: loop invariants written as pragmas, checked by GNATprove. Why3: WhyML has its own specification and programming language. F\*: proofs are written with the program. Lean not checked | Dafny reference manual; Verus paper (arXiv:2303.05491); AdaCore loop-invariant blog; Why3 publication and package page; F\* tutorial |
| EV-2 | None of the six restricts the verified program to a granted capability set by default | **Open.** F\* has a lattice of tracked effects (the ICFP 2015 paper names PURE, DIV, STATE and EXN). No source found shows default capability restriction in any of the six, but a search not finding it is not proof | *Semantic Purity and Effects Reunited in F\** (ICFP 2015); F\* tutorial, divergence chapter |
| EV-3 | Verus implementations are Rust and are borrow-checked; the others compile or extract to other languages | **Confirmed.** Verus relies on Rust's borrow checker, and ghost and spec code is exempt from linearity checks. Dafny targets on the current manual page: C#, Java, JavaScript, Go, Python, C++ (page truncated, list may be incomplete). F\* extracts to OCaml and F#, and to C through Low\*/KaRaMeL. Why3 extracts to OCaml and C | Verus paper; Dafny reference manual; F\* tutorial; Why3 package page |
| EV-4 | Existing tooling is oriented to human use; list agent wrappers | **Partly refuted.** A community Dafny verifier MCP server exists (listed as released January 2025, small uptake). Several community Lean MCP servers exist. No Verus MCP server turned up in one search | playbooks.com listing for `namin/dafny-verifier`; Lean MCP listings on glama.ai and readthedocs |
| EV-5 | Each system has an escape hatch; `.opq` has none | **Confirmed for both halves.** Dafny: `assume`, `{:axiom}`, a bodiless method whose `ensures` are assumed, and a `/noCheating` option (manual text for `assume` itself not read). Verus: `assume`, `admit`, `external_body`, `assume_specification`; `--no-cheating` flag (verus PR #1499, 2025; a third-party report says it does not reject `axiom` functions). F\*: `assume` and `admit`. SPARK: `pragma Assume` ("a variant of Assert that GNATprove does not check", with a justification string). Lean: `sorry`, reported as `sorryAx`, with a warning. `.opq`: no such form in the grammar (claim B). Why3 not checked | Dafny manual; Verus PR #1499; F\* tutorial; AdaCore SPARK User's Guide; Lean reference notes; `docs/spec/components/cprime-syntax.md` |
| EV-6 | What the benchmark and related papers report | **Checked at abstract level.** DafnyBench: over 750 programs, best 68% at publication. Clover (arXiv:2310.17807): a consistency check across code, docstring and annotations; reported acceptance figures differ between sources, so none is quoted. AlphaVerus (arXiv:2412.06176; Aggarwal, Parno, Welleck): self-improving Dafny-to-Verus translation with verifier feedback; no figures in the abstract. Marmaragan (arXiv:2502.07728; Cramer, McIntyre): GPT-4o produced correct SPARK annotations for 50.7% of its benchmark cases. ExVerus (arXiv:2603.25810; Yang et al., March 2026): counterexample-guided proof repair for Verus, improving accuracy, robustness and token efficiency over a prompting baseline; the abstract gives no numbers | The arXiv pages |

**Still not confirmed.** These items need the primary source and were not read:

- Dafny's own manual text on `assume` and `{:axiom}` (only headings and a note on bodiless methods were retrieved).
- Why3's escape hatches, and Lean's compiled-code behaviour with `sorry` (Lean's `#eval` refusal and warning were found, the C backend was not).
- The Verus guide on `assume` and `admit` (the guide URL tried returned 404; the facts above come from a pull request and secondary pages).
- Whether F\* can express "no ambient authority" for agent-written code (EV-2).
- The body of every paper above. Only abstracts were read, so no figure from a paper's results table is quoted.
- The `.opq` prelude: the grammar allows `(path expr...)` for "prelude fn, spec call, constructor". The prelude's function list was not read, so the claim that `.opq` has no escape hatch holds for the grammar and not yet for the prelude.

**Read before the essay is submitted.** ExVerus is the closest prior work to C′'s counterexample loop and to the witness traces of RFC 03. Marmaragan, Verina (arXiv:2505.23135) and VeriEquivBench (arXiv:2510.06296) are also relevant to the related-work section. The field is moving quickly, so the essay should expect close prior work on agents, counterexamples and verifiers.

## Dependencies

- Rust toolchain (2024 edition) and Cargo, on every machine that builds C′ or a project-specific runtime.
- Z3 (through the `z3` crate) and LLVM (through `inkwell`), versions per OQ-16.
- Crates: `salsa`, `chumsky`, `blake3`, `z3`, `cranelift-codegen`, `cranelift-frontend`, `cranelift-module`, `cranelift-jit`, `inkwell`, `arbitrary`, `tokio`, `serde`, `serde_json`.
- An MCP- or JSON-RPC-capable AI agent client.
- The 50-task H1 corpus and the Phase 5 benchmark suite (to be built).
- Third-party provider crates chosen by each project.

## Rollout and Rollback

**Rollout** follows the concept's phased roadmap; each phase ends at a testable exit criterion, not a date. Nothing is JIT-compiled before reference counting exists.

| Phase | Scope | Exit criterion |
|---|---|---|
| 1 — Front end | Workspace; `cprime-syntax` for both surfaces; namespace table and hashing; `cprime-types` with sum types, exhaustive `match`, monomorphisation; `cprime-effects` with termination check | The concept example parses, type-checks and passes effect checks; an undeclared capability yields `UnauthorizedEffect`; a `match` on `Result` missing `Err` yields `NonExhaustiveMatch` |
| 2 — IR and memory | Lowering to C′-IR; Perceus RC and FBIP; IR interpreter as oracle | Interpreter runs the example; allocation-counting harness shows zero leaks across the corpus |
| 3 — JIT and daemon | Cranelift JIT; `cprimed` with `get_context`, `check_impl` and MCP tools; persistence and layout; provider SDK and runtime build; four built-ins on the SDK; runtime limits | T0/T1 meet targets; JIT matches interpreter; H1 measured and reported; a sample third-party provider works end to end |
| 4 — Verification | Z3 lowering; obligations; `submit_impl`; evidence levels | The concept's rejection is reproduced; the corrected implementation reaches `proved`; the revision-1 contract yields `ReachableTrap` |
| 5 — Fuzzing and release | Fuzzing harness; `assurance tested`; LLVM AOT; benchmarks | Performance target measured and reported; AOT output matches the interpreter on the corpus |

**Decisions each phase depends on:** Phase 1 — ADR-023, ADR-024, the grammar part of ADR-026, OQ-29. Phase 2 — ADR-022. Phase 4 — ADR-025, ADR-026. The `/spec` gates for each phase are listed under *Must be settled in `/spec` before a phase starts*.

**Rollback.** Greenfield, with no shared database or hosted service. Rolling back a compiler release means reinstalling the previous `cprimec`/`cprimed` and restarting the daemon, which rebuilds its cache from disk. An unwanted acceptance is reverted in the user's version control because each acceptance is one file. If OQ-14 introduces format or hash versioning, its rollback rules are fixed in `/spec`.
