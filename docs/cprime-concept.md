# C′ — AI-native compiler — concept

| Field | Value |
|---|---|
| Status | Draft — revision 4 |
| Date | 2025-12-20 |
| Supersedes | Revision 3 |
| Next step | HLD (`docs/design/cprime/`, revision 0.12.0) and the approved spec set (`docs/spec/`) exist. Run `/breakdown` for RFC 07, then implement the Generation Playbook (`docs/spec/phase-5-playbook.md`) one step at a time with `codegen`. |

This is the technical and architectural concept for **C′ (`cprimec`)**, a verification-first systems compiler written in Rust, in which humans own the contracts and AI agents write the implementations.

---

## 1. Summary and principles

**C′** is a systems programming language and compiler daemon for a *human-specifies, AI-implements* workflow.

1. **Humans own `.cprime` files.** Behavioural contracts, type definitions, capability grants, termination and assurance policy. A person writes them, or an AI agent drafts them from a plan the person has agreed (in a spec-driven process such as CSPA) and the person reviews and approves the draft before anything is implemented against it. A contract is short and declarative, so reviewing it costs far less effort than reviewing an implementation. `cprimed` does not distinguish who wrote a `.cprime` file and offers no request for writing one (HLD ADR-046).
2. **AI agents write `.opq` files.** Compact S-expression implementations that bind to a `.cprime` spec. Humans are not expected to review them line by line.
3. **No borrow checker.** Memory safety comes from immutable value semantics plus compile-time Perceus reference counting. The agent never annotates lifetimes or ownership.
4. **Effects are explicit.** Every side effect is an operation of a capability. A function may only perform operations of capabilities its spec lists in `uses`. This is checked at compile time.

### What C′ guarantees, and what it does not

C′ guarantees that an accepted implementation (a) type-checks, (b) performs no effect outside its spec's `uses` list, (c) assuming the spec's `requires`, establishes every `ensures` clause and the unreachability of every trap, and establishes the `requires` of every spec it calls, at the evidence level the spec's `assurance` policy demands, and (d) terminates, unless the spec grants `Diverge`, provided every provider call returns. (d) does not exclude a trap on a configured CPU-time, memory or call-depth limit (§5.2).

These guarantees are conditional. A proof assumes the contracts of the specs the implementation calls (HLD ADR-033) and the verifier's model of each provider operation (§5.4). It holds for the running program only if those callees and providers meet them and the trusted computing base (§5.2) is correct. A `requires` that no input satisfies makes every obligation hold trivially; v1 does not detect this (§5.2).

C′ guarantees **nothing about behaviour the contract does not state.** A property absent from `ensures` is unverified. Because of this, "humans never read the `.opq`" is safe only to the extent that the contract is complete. C′ also does not check a contract against the plan it was drafted from; when an agent drafts the contract, the person's review is the only check on that step. The compiler reports result fields that no `ensures` clause mentions (`UnconstrainedOutput`, §6.4) and rejects the implementation unless the spec waives those fields. This is an omission check, not a completeness guarantee: a field mentioned only by a clause that adds no information, such as `result.n >= 0` for a `U64`, passes it.

---

## 2. Scope of v1

### Goals

- Parse, type-check and effect-check `.cprime` and `.opq` sources.
- Lower implementations to an SSA IR, insert reference-counting operations, JIT-compile with Cranelift.
- Discharge contracts with Z3; fall back to property-based fuzzing where policy allows.
- Serve all of the above over JSON-RPC 2.0 and as MCP tools from a long-running daemon.
- AOT release builds through LLVM.
- Use several CPU cores in release binaries without shared memory: `par_map` over a `Vec` (HLD ADR-044), and long-running servers in which the runtime spawns a short-lived, share-nothing task per request to run a handler spec (HLD ADR-051).

### Non-goals for v1

| Non-goal | Consequence for v1 |
|---|---|
| Region inference | All heap values are reference counted. Revisit after Phase 5 benchmarks. |
| Shared-memory multi-threading | No C′ value is shared between threads, so RC counts are non-atomic. Release binaries use worker threads only for `par_map` and for per-request server tasks, each with its own heap (HLD ADR-044, ADR-051). Program-spawned tasks with channels come after v1 (HLD ADR-040). |
| Mutable references | None in the language. In-place mutation happens only through FBIP reuse (§5.3). |
| FFI from `.opq` | `.opq` cannot call foreign code. Foreign code exists only inside host capability providers (§5.2). |
| User-defined effect handlers in `.opq` | Handlers are bound at the program entry point in `.cprime`. |
| Side-channel resistance | Timing and cache behaviour are not constrained (§5.2, residual risks). |
| Built-in network and filesystem providers | None ship with C′; a project may supply its own through the provider SDK (§5.2, OQ-10). |
| Dynamically loaded providers | Providers are statically linked into a project-specific runtime (§5.2, OQ-10). |

---

## 3. System architecture

The compiler is written in **Rust (2024 edition)** and runs as a long-lived daemon (`cprimed`) with a thin CLI (`cprimec`). Agents work against in-memory state over RPC, so the edit–check loop does not round-trip through the filesystem.

### 3.1 Cargo workspace

```text
cprime/
├── Cargo.toml
└── crates/
    ├── cprime-core/        # Namespace table, content hashes, IR definitions, Salsa database
    ├── cprime-syntax/      # Parsers for .cprime and .opq, with error recovery
    ├── cprime-types/       # Bidirectional type checker
    ├── cprime-effects/     # Capability/effect inclusion checker, termination check
    ├── cprime-memory/      # Perceus RC insertion and FBIP reuse
    ├── cprime-verify/      # Z3 lowering, proof obligations, fuzzing harness
    ├── cprime-codegen/     # Cranelift JIT (iteration) and LLVM AOT (release)
    ├── cprime-daemon/      # JSON-RPC 2.0 server and MCP tool wrapper
    └── cprime-provider/    # Provider SDK: the interface every host provider implements (OQ-10)
```

`cprime-types` is split from `cprime-syntax` because Phase 1 delivers it as a separate exit criterion (§8).

### 3.2 Dependencies

| Purpose | Crates |
|---|---|
| Incremental computation | `salsa` |
| Parsing with recovery | `chumsky` (OQ-1) |
| Content hashing | `blake3` |
| SMT | `z3` |
| JIT | `cranelift-codegen`, `cranelift-frontend`, `cranelift-module`, `cranelift-jit` |
| AOT | `inkwell` (LLVM) |
| Fuzzing input generation | `arbitrary`, driving JIT-compiled code inside `cprimed` (OQ-2) |
| Daemon / IPC | `tokio`, `serde`, `serde_json` |

LLM traffic is JSON text. A binary encoding would shrink bytes on the wire but not the tokens a model reads, so none is included.

### 3.3 Latency budget

Targets are p95, measured on a single implementation of at most 200 IR instructions, with the Salsa database warm.

| Tier | Work | Target | Runs in |
|---|---|---|---|
| T0 | Parse, resolve names, type-check, effect-check, termination check | ≤ 5 ms | `check_impl`, `submit_impl` |
| T1 | IR lowering, Perceus pass, Cranelift JIT | ≤ 20 ms | `check_impl`, `submit_impl` |
| T2 | SMT verification | ≤ 250 ms per proof obligation; ≤ 2 s per submission (configurable) | `submit_impl` |
| T3 | Fuzzing, only where T2 returns `unknown` or times out *and* the spec's `assurance` is `tested` | 10,000 iterations, ≤ 5 s (configurable) | `submit_impl` |

The agent iterates on `check_impl` (T0–T1, sub-25 ms) and calls `submit_impl` once the fast checks pass.

### 3.4 Persistence

The daemon is a cache, not the source of truth.

- `.cprime` and `.opq` files on disk are authoritative. On start-up, `cprimed` loads them into the Salsa database.
- `check_impl` never writes to disk.
- When `submit_impl` accepts an implementation, `cprimed` writes it to its `.opq` file atomically (write to a temporary file, then rename) before replying. Version control therefore records every accepted implementation.
- Rejected submissions are not persisted.
- On-disk layout (OQ-3): one `.opq` file per implementation, in a directory named after the module, beside the module file. `auth.cprime` → `auth/generate_token.opq`.

---

## 4. Language

A program has two surfaces that resolve to the same semantic graph.

### 4.1 Human surface — `.cprime`

Declarative: types, capabilities, contracts and policy.

```text
// auth.cprime
module Auth

// A capability is a named set of effect operations.
// `deterministic` lets the verifier model the operation as a mathematical function (§5.4).
capability CryptoCap {
    deterministic hmac_sha256(key, msg Bytes) Hash
}
// No network or filesystem capability is declared, so no implementation in this module can perform one.

struct SessionToken {
    user_id    U64
    expires_at U64
    sig        Hash
}

// assurance proved is the default, so it is not written.
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

The syntax is in the C and Go family (HLD ADR-050): a name comes before its type, a newline ends a statement, every body is in braces, and each clause is one line starting with its keyword. `ensures` refers to the result as `result`. `Hash` is defined in the prelude as `type Hash = FixedBytes[32]`. `u64_be` (big-endian encoding) and `++` (byte concatenation) are pure prelude functions. A grammar sketch is in Appendix A.

### 4.2 AI surface — `.opq`

An `.opq` implementation names the spec it implements and binds its parameters positionally. It does **not** restate parameter types, the return type or effects; those come from the spec, so the two cannot disagree.

```lisp
;; implements Auth::generate_token
(impl Auth::generate_token [uid now secret]
  (let [exp (+ now 3600)
        msg (++ (u64_be uid) (u64_be exp))
        sig (perform CryptoCap::hmac_sha256 secret msg)]
    (SessionToken uid exp sig)))
```

What makes `.opq` compact is its structure (no imports, no type restatement, no indentation rules), not abbreviated identifiers. Local binding names are free-form; spec, type and capability names are always fully qualified.

**Hypothesis H1 (unmeasured):** for the same contracts, `.opq` implementations use at least 40% fewer model tokens than idiomatic Rust, with no drop in first-submission acceptance rate. Phase 3 measures this (§8).

**Hypothesis H2 (unmeasured):** the Go-like `.cprime` syntax is at least as easy for people to read and write as the earlier Rust-like syntax. On the two concept specs it uses 160 tokens instead of 196 and 62 punctuation tokens instead of 94; that is a proxy, not a measurement. Phase 3 measures it with 8–10 developers new to C′. Each reviews 10 contracts in both syntaxes, in a counterbalanced order, each with one planted defect, and then writes contracts from plain-English descriptions. Recorded: defects found, time taken, and parse errors on the first attempt. Pass condition: the new syntax is not worse on any of them (HLD `10-risks.md`).

### 4.3 Core semantics

| Concern | v1 rule |
|---|---|
| Integer arithmetic | Checked. `+ - *` trap on overflow; `/` and `%` trap on division by zero. The verifier must prove no trap is reachable under `requires`. Explicit `wrapping_add`, `wrapping_sub`, `wrapping_mul` exist for intentional modular arithmetic. |
| Indexing | `(index v i)` traps when `i >= len(v)`; the verifier must prove it unreachable. |
| Sum types | User-declared, for example `enum Shape { Circle { r U64 }; Square { side U64 } }`. `.opq` constructs a variant as `(Circle 5)` and destructures with `(match e [(Circle r) body] [(Square s) body])`. A `match` must cover every variant (`NonExhaustiveMatch`). Generic type definitions are monomorphised before IR lowering (OQ-4). |
| Failure | No exceptions. Recoverable failure is returned as `Result[T, E]`, an ordinary sum type defined in the prelude: `enum Result[T, E] { Ok { value T }; Err { error E } }` (OQ-4). Traps abort the program. |
| Termination | Implementations must terminate. Recursion must carry a `(decreases e)` clause whose measure the verifier proves strictly decreasing over `U64`. A spec may grant `Diverge` to opt out. |
| Mutability | None. All values are immutable; see §5.3 for in-place reuse. |

---

## 5. Compiler subsystems

### 5.1 Content-addressed semantic graph (`cprime-core`)

- Each definition (type, spec, capability, implementation) is identified by a `DefId`: the Blake3 hash of its canonical serialisation, in which references to other definitions are replaced by their `DefId`s and local bindings are replaced by de Bruijn indices. Renaming a local binding therefore does not change the hash.
- A **namespace table** maps qualified names (`Auth::generate_token`) to `DefId`s. RPC calls use names; the daemon resolves them through this table.
- **Context queries.** `get_context` for a spec returns only the definitions transitively reachable from it: types, capability signatures and prelude functions referenced in its signature and contract. The agent does not read files or write imports.

### 5.2 Capabilities and effects (`cprime-effects`)

**One model.** A capability is a named set of effect operations. The effect set of a function type is a set of capabilities. There are no capability tokens as runtime values in v1.

**Rules.**

1. There is no ambient authority: no global filesystem, network, clock, randomness or environment access.
2. Every operation with an observable side effect is an effect operation of some capability.
3. `cprime-effects` computes the transitive effect set of each implementation (its own `perform`s plus the declared effects of everything it calls) and rejects any implementation whose set is not a subset of its spec's `uses` list (`UnauthorizedEffect`).
4. `Diverge` is a built-in capability. A function without it must pass the termination check (§4.3).
5. `.opq` cannot install handlers and cannot call foreign code.

**Handlers and the trust boundary.** Capability operations are implemented by **host providers**: Rust code compiled into the C′ runtime. A program's `.cprime` entry point binds each capability it uses to a provider (`bind CryptoCap = host::Crypto`). A binding is valid only if the provider implements every operation of the capability with an identical signature. Providers are the only place foreign code runs.

**Built-in providers (OQ-5):** `host::Crypto`, `host::Clock`, `host::Random`, `host::Log`. None offers network or filesystem access. The operations each exposes are defined during `/spec`.

**Provider SDK (OQ-10).** Projects reuse existing Rust crates by writing their own providers:

- A provider is a Rust crate depending on `cprime-provider`. It implements one capability and declares, per operation, its name, C′ signature and `deterministic` flag. The four built-ins are written on the same SDK.
- Types crossing the boundary are limited to `Bool`, `U8`, `U64`, `I64`, `Bytes`, `FixedBytes[N]`, `Vec` of these, and `Result[T, E]` over these. Conversion is a copy of plain data.
- A project manifest lists the provider crates. `cprimec` builds a project-specific runtime with Cargo, linking them statically; `cprimed` uses the same build for JIT checks and fuzzing, so development and release run identical provider code. A rebuild is needed only when the provider list or a provider crate changes. Manifest name, location and format are defined during `/spec`.
- `bind` is accepted only if the provider implements every operation of the capability with an identical signature and the same `deterministic` flag.
- Every provider joins the trusted computing base. Each release build reports a **provider inventory**: every linked provider crate with its exact version from `Cargo.lock`.
- The fuzzing harness calls each `deterministic` operation twice with the same inputs and reports a difference as `NonDeterministicProvider` (§6.4). This catches some false determinism claims, not all.

**Trusted computing base:** `cprimec`/`cprimed`, Z3, Cranelift, LLVM, and the host providers. A bug in any of these can break the guarantees in §1.

**Residual risks (not addressed in v1):**

- Timing and cache side channels.
- Resource exhaustion: termination is proved, but a terminating program can still be slow or memory-heavy. Limits on CPU time and memory are set in `cprimed`/`cprimec` configuration, not in `.cprime`; exceeding either aborts the program (OQ-6). Configuration keys are defined during `/spec`.
- An operation declared `deterministic` whose provider is not deterministic makes proofs about it unsound. Provider determinism is asserted by the provider author, not checked.
- Contract incompleteness (§1). `UnconstrainedOutput` catches omitted fields, not uninformative clauses.
- Vacuous and inconsistent contracts. A `requires` no input satisfies makes every obligation trivially `unsat`; contradictory `ensures` clauses make every implementation fail. Neither is diagnosed.
- Conditional evidence across calls. A caller's evidence level assumes its callees' contracts (HLD ADR-033), whether each callee is proved, only tested or not yet implemented. The acceptance header records the caller's own evidence level only.
- Information flow. Capabilities limit which effects an implementation may perform, not what data reaches them: a permitted `Log` operation can receive a secret. C′ provides no confidentiality or noninterference guarantee.
- Provider authority. The provider inventory lists the trusted code; it does not show that a provider's native code lacks authority beyond its declared operations. Providers are trusted, not verified or isolated.
- Readability of `.opq`. There is no human-readable rendering of `.opq` and no source map from it back to the contract; witness traces and trap-site tables are the only aids for diagnosis and independent review.

### 5.3 Memory model (`cprime-memory`)

- **Value semantics.** All values are immutable.
- **Perceus reference counting.** After IR lowering, `cprime-memory` computes liveness for every heap-typed SSA value (`Bytes`, `FixedBytes`, `Vec`, structs containing these) and inserts `RcDup` before each additional use and `RcDrop` after the last use. Scalars (`Bool`, `U8`, `U64`, `I64`) are not counted.
- **FBIP reuse.** Where a value is dropped and a value of the same layout is constructed afterwards in the same block, the pass emits `DropReuse` and passes the resulting token to the construction. At run time, if the count was 1, the memory is reused in place; otherwise a fresh allocation is made.
- **No cycles.** With no mutable references and no laziness, a value can only reference values that existed before it, so reference graphs are acyclic and RC reclaims all memory. In-place reuse does not break this: it happens only when the count is 1, so no other value can observe the old contents.
- **Share-nothing.** Counts are non-atomic (§2): worker threads (`par_map`, server tasks) each have their own heap, and values cross between heaps only by copy.
- **Performance target (unmeasured):** release builds within 2× the run time of equivalent hand-written Rust on the Phase 5 benchmark suite.

### 5.4 Verification (`cprime-verify`)

**Proof obligations.** For each accepted-candidate implementation, `cprime-verify` lowers the spec's `requires`, the implementation's IR and the spec's `ensures` to Z3 and generates one obligation per:

- `ensures` clause;
- potential trap site (overflow, division by zero, index out of bounds);
- `decreases` clause.

Z3 searches for an input that satisfies `requires` and violates the obligation. A model found is returned as a counterexample. The model is a counterexample to the verifier's abstract model, not necessarily to the running program: it may give a `deterministic` operation's uninterpreted function a value no real provider returns. A rejection without a model — `unknown` or a timeout — carries no counterexample.

**Modelling effects.**

| Operation kind | Model |
|---|---|
| `deterministic` operation | Uninterpreted function of its arguments (so the `sig` clause in §4.1 is provable by congruence) |
| Any other operation | A fresh unconstrained symbolic value per call |

**Evidence levels.**

| Level | Meaning |
|---|---|
| `proved` | Z3 returned `unsat` for every obligation |
| `tested` | At least one obligation was `unknown` or timed out; fuzzing ran the configured iteration budget without finding a violation |
| `none` | Neither of the above |

**Assurance policy.** Each spec declares `assurance proved` (default) or `assurance tested`. `submit_impl` accepts an implementation only if its evidence level meets the policy. Under `assurance proved`, an `unknown` result is a rejection (`VerificationInconclusive`), never a silent downgrade.

**Fuzzing harness (OQ-2).** A custom generator inside `cprimed` builds structured inputs with the `arbitrary` crate, keeps only those that satisfy `requires` (rejection sampling in v1), and calls the JIT-compiled implementation directly in memory. `deterministic` operations call the real host provider. Other operations are replaced by stubs that return generator-chosen values, so fuzzing never performs real I/O.

**Omission check (OQ-7; the spec set calls it the completeness check).** When the result type is a struct, every field must be mentioned as `result.<field>` by at least one `ensures` clause or be waived by name in the spec's `unconstrained` clause. A field that is neither yields an `UnconstrainedOutput` error, and the implementation is rejected. The waiver records that the human accepted the gap knowingly:

```text
spec make_session(user_id U64) Session {
    uses Random
    unconstrained nonce
    ensures result.user_id == user_id
}
```

Completeness rules for sum-typed results are defined during `/spec`.

---

## 6. AI–compiler protocol

### 6.1 JSON-RPC 2.0 methods

| Method | Tiers | Writes disk | Purpose |
|---|---|---|---|
| `get_context` | — | No | Return the reachable context of a spec (§5.1) |
| `check_impl` | T0–T1 | No | Fast feedback loop |
| `submit_impl` | T0–T3 | Yes, on acceptance | Verify and accept |

The same three operations are exposed as MCP tools named `cprime_get_context`, `cprime_check_impl` and `cprime_submit_impl`. Each tool's arguments are the `params` object of the matching JSON-RPC method (OQ-8).

### 6.2 Example request

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "submit_impl",
  "params": {
    "spec": "Auth::generate_token",
    "source": "(impl Auth::generate_token [uid now secret] ...)"
  }
}
```

### 6.3 Example response

The agent wrote `(+ now 1000)` instead of `(+ now 3600)`. Timing values are illustrative.

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "status": "rejected",
    "evidence": "none",
    "timings_us": { "t0": 1900, "t1": 8400, "t2": 41000 },
    "diagnostics": [
      {
        "kind": "ContractViolation",
        "severity": "error",
        "clause": "ensures: result.expires_at == now + 3600",
        "node": 4,
        "counterexample": {
          "inputs": { "user_id": 1, "now": 100, "secret": "<32 bytes>" },
          "expected": { "result.expires_at": 3700 },
          "actual":   { "result.expires_at": 1100 }
        }
      }
    ]
  }
}
```

Compile errors (T0/T1) use the JSON-RPC `result` with `status: "rejected"`, not the JSON-RPC `error` object. The `error` object is reserved for protocol faults (unknown method, malformed parameters, daemon failure).

### 6.4 Diagnostic kinds

| Kind | Tier | Severity | Trigger |
|---|---|---|---|
| `ParseError` | T0 | error | Source does not match the grammar |
| `UnknownName` | T0 | error | Name not in the namespace table |
| `ArityMismatch` | T0 | error | `impl` binds a different number of parameters than the spec declares |
| `TypeMismatch` | T0 | error | Type checker failure |
| `UnauthorizedEffect` | T0 | error | Effect set not a subset of `uses` |
| `MissingDecreases` | T0 | error | Recursion without `decreases` and without `Diverge` |
| `ContractViolation` | T2/T3 | error | Counterexample to an `ensures` clause |
| `ReachableTrap` | T2/T3 | error | Counterexample reaching overflow, division by zero or out-of-bounds |
| `NonDecreasing` | T2 | error | `decreases` measure not proved to decrease |
| `VerificationInconclusive` | T2 | error | `unknown`/timeout under `assurance proved` |
| `NonExhaustiveMatch` | T0 | error | A `match` does not cover every variant of its sum type |
| `UnconstrainedOutput` | T0 | error | Result field neither mentioned by any `ensures` clause nor waived by `unconstrained` |
| `NonDeterministicProvider` | T3 | error | A `deterministic` provider operation returned different results for identical inputs |
| `InvalidServer` | T0 | error | A server's handler or transport breaks the server rules: more than one parameter, a `requires` clause, `Diverge`, a provider that is not thread-safe, or request/response records that do not match the transport (HLD ADR-051) |

---

## 7. Intermediate representation (C′-IR)

Defined in `cprime-core/src/ir.rs`. SSA with block arguments in place of phi nodes, matching Cranelift's model. The content hash (§5.1) is computed over the canonical AST, not the IR. Generic definitions are monomorphised before lowering, so the IR has no type parameters (OQ-4).

```rust
use std::collections::BTreeSet;

/// Interned identifier.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct Symbol(pub u32);

/// Blake3 content hash of a definition.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct DefId(pub [u8; 32]);

/// Interned capability; effect sets are sets of these.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct CapId(pub u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ValueId(pub u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct BlockId(pub u32);

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum Type {
    Bool,
    U8,
    U64,
    I64,
    Bytes,
    FixedBytes(u32),
    Vec(Box<Type>),
    /// Field layout is looked up from the struct definition.
    Struct(DefId),
    /// Variants and their field layouts are looked up from the sum-type definition.
    Sum(DefId),
    Fn {
        params: Vec<Type>,
        ret: Box<Type>,
        effects: BTreeSet<CapId>,
    },
}

#[derive(Debug, Clone)]
pub enum Const {
    Bool(bool),
    U8(u8),
    U64(u64),
    I64(i64),
    Bytes(Vec<u8>),
}

#[derive(Debug, Clone, Copy)]
pub enum BinOp {
    // Checked: lowering emits a trap edge on overflow or division by zero.
    Add, Sub, Mul, Div, Rem,
    WrappingAdd, WrappingSub, WrappingMul,
    Eq, Ne, Lt, Le, Gt, Ge,
    And, Or,
    BytesConcat,
}

#[derive(Debug, Clone)]
pub enum Inst {
    Const(Const),
    Bin { op: BinOp, lhs: ValueId, rhs: ValueId },
    Call { func: DefId, args: Vec<ValueId> },
    Perform { cap: CapId, op: Symbol, args: Vec<ValueId> },
    /// `reuse` is a token from `DropReuse`, inserted by `cprime-memory`.
    MakeStruct { ty: DefId, fields: Vec<ValueId>, reuse: Option<ValueId> },
    GetField { value: ValueId, index: u32 },
    /// `tag` is the variant's position in its sum-type definition.
    MakeVariant { ty: DefId, tag: u32, fields: Vec<ValueId>, reuse: Option<ValueId> },
    /// Valid only in a block reached from a `Switch` case for `tag`.
    GetVariantField { value: ValueId, tag: u32, index: u32 },
    Len(ValueId),
    /// Bounds-checked: lowering emits a trap edge when out of range.
    Index { vec: ValueId, index: ValueId },
    // Inserted by `cprime-memory`; never emitted by the front end.
    RcDup(ValueId),
    RcDrop(ValueId),
    DropReuse(ValueId),
}

#[derive(Debug, Clone, Copy)]
pub enum TrapKind {
    Overflow,
    DivisionByZero,
    IndexOutOfBounds,
    Unreachable,
}

#[derive(Debug, Clone)]
pub enum Terminator {
    Return(ValueId),
    Jump { target: BlockId, args: Vec<ValueId> },
    Branch {
        cond: ValueId,
        then_target: BlockId,
        then_args: Vec<ValueId>,
        else_target: BlockId,
        else_args: Vec<ValueId>,
    },
    /// Dispatch on a sum value's tag; `cases[tag]` is the target for that variant.
    /// Exhaustive by construction: `cases.len()` equals the variant count.
    Switch {
        scrutinee: ValueId,
        cases: Vec<(BlockId, Vec<ValueId>)>,
    },
    Trap(TrapKind),
}

#[derive(Debug, Clone)]
pub struct Block {
    pub params: Vec<(ValueId, Type)>,
    pub insts: Vec<(ValueId, Inst)>,
    pub term: Terminator,
}

#[derive(Debug, Clone)]
pub struct Function {
    pub def: DefId,
    pub ty: Type, // always Type::Fn
    pub entry: BlockId,
    pub blocks: Vec<Block>,
}
```

---

## 8. Roadmap

Ordered so that nothing is JIT-compiled before reference counting exists. Each phase ends at a testable exit criterion, not a date.

| Phase | Scope | Exit criterion |
|---|---|---|
| 1 — Front end | Workspace; `cprime-syntax` (with `chumsky`) for both surfaces; namespace table and content hashing; `cprime-types` including sum types, exhaustive `match` and monomorphisation; `cprime-effects` including termination check | The §4.1/§4.2 example parses, type-checks and passes effect checks; a variant that performs an undeclared capability yields `UnauthorizedEffect`; a `match` on `Result` missing `Err` yields `NonExhaustiveMatch` |
| 2 — IR and memory | Lowering to C′-IR; Perceus RC and FBIP in `cprime-memory`; an IR interpreter as a reference oracle | Interpreter runs the example; a test harness counting allocations shows zero leaks across the test corpus |
| 3 — JIT and daemon | Cranelift JIT; `cprimed` with `get_context`, `check_impl` and the `cprime_*` MCP tools; persistence and on-disk layout (§3.4); `cprime-provider` SDK and project-specific runtime build; built-in providers `host::Crypto`, `host::Clock`, `host::Random`, `host::Log` written on the SDK; runtime limits in configuration | T0 and T1 meet §3.3 targets; JIT output matches the interpreter on the corpus; H1 measured on the OQ-9 corpus of 50 tasks and reported, whether confirmed or not; H2 measured and reported, whether confirmed or not; a sample third-party provider wrapping an existing Rust crate works end to end (bind, check, submit, release build, provider inventory) |
| 4 — Verification | Z3 lowering; obligations for `ensures`, traps and `decreases`; `submit_impl`; evidence levels | The §6.3 rejection is reproduced; the corrected implementation reaches `proved`; the revision-1 contract (without the `now` bound) yields `ReachableTrap` |
| 5 — Fuzzing and release | Fuzzing harness; `assurance tested`; LLVM AOT; benchmark suite; server runtime and sample HTTP transport (HLD ADR-051) | §5.3 performance target measured and reported, whether met or not; the sample server serves concurrent requests on all workers, survives a trapping request, runs 1,000,000 requests without stack or memory growth, and shuts down cleanly; programs that are not servers stay within 2% of their run time without the feature |

---

## 9. Decisions

These entries began as open questions. All ten were approved on 2025-12-20, and the sections listed under *Affects* reflect them. An entry marked **Proposed** awaits approval; the body does not reflect it until it is **Decided**. The `OQ-n` ids are kept because the body cites them.

| Id | Question | Decision | Rationale | Affects | Needed by | Status |
|---|---|---|---|---|---|---|
| OQ-1 | `chumsky` or `winnow` for parsing | `chumsky` | Built-in error recovery lets one submission report every parse error, so the agent fixes more per round trip. Parse time is a small share of the T0 budget. | §3.2 | Phase 1 start | Decided |
| OQ-2 | Fuzzing engine: an embedded library, or a custom generator driving the JIT | Custom generator driving the JIT, using the `arbitrary` crate for structured inputs | Off-the-shelf fuzzers run a separate binary over raw bytes. C′ needs inputs that satisfy `requires`, stubbed effects, and execution in memory inside `cprimed`. | §3.2, §5.4 | Phase 5 start | Decided |
| OQ-3 | On-disk layout of `.opq` files relative to `.cprime` files | One file per implementation, in a directory named after the module, beside the module file: `auth.cprime` → `auth/generate_token.opq` | Each acceptance is one atomic file write (§3.4) and one self-contained git diff. | §3.4 | Phase 3 start | Decided |
| OQ-4 | Shape of the recoverable-failure sum type | General user-declared sum types; `Result[T, E]` defined in the prelude | Programs need sum types beyond failure; `Result` then needs no special case. Consequence: the IR (§7) gains sum-type construction and a tag-dispatch terminator, and the grammar (Appendix A) gains sum-type declarations. | §4.3, §7, Appendix A | Phase 1 start | Decided |
| OQ-5 | Host provider catalogue for v1 and how providers are registered | Four host providers — `host::Crypto`, `host::Clock`, `host::Random`, `host::Log` — with no network or filesystem. Providers are Rust implementations compiled into the runtime; no dynamic loading. | Keeps the trusted computing base (§5.2) small while covering the examples, the H1 corpus and the benchmark suite. | §2, §5.2, §8 | Phase 3 start | Decided; catalogue opened by OQ-10 |
| OQ-6 | Runtime limits on CPU time and memory, and whether they are expressed in `.cprime` | Set in `cprimed`/`cprimec` configuration, not in `.cprime`. Exceeding a limit aborts the program. | Termination is already proved (§4.3), so limits guard only against slow or memory-heavy programs; a language feature for this adds grammar and verifier work for little v1 benefit. | §5.2, §8 | Phase 3 start | Decided |
| OQ-7 | Whether `UnconstrainedOutput` blocks acceptance, and whether a spec may waive it per field | Blocking by default; a spec may waive named fields with an `unconstrained [<field>, …]` clause | The human must accept a gap in the contract explicitly, which is consistent with verification-first acceptance. Consequence: `UnconstrainedOutput` severity becomes `error`, and the grammar gains the waiver clause. | §5.4, §6.4, Appendix A | Phase 4 start | Decided |
| OQ-8 | MCP tool names and their argument schemas | Names: `cprime_get_context`, `cprime_check_impl`, `cprime_submit_impl`. Arguments mirror the JSON-RPC `params` (§6.2). | One vocabulary across both interfaces reduces documentation and test surface. | §6.1 | Phase 3 start | Decided |
| OQ-9 | Contract corpus and Rust baseline used to measure H1 | 50 small, well-known algorithmic and systems tasks. The same model writes both the `.opq` and the Rust implementation from the same prompt. Measure tokens with that model's tokeniser, and first-submission acceptance rate. | Using one model and one prompt for both languages isolates the effect of the language from the effect of the model. | §4.2, §8 | Phase 3 start | Decided |
| OQ-10 | How users reuse existing Rust crates, given that `.opq` has no FFI (§2) | A **provider SDK**: users write their own providers as Rust crates, statically linked into a project-specific runtime. Detail in §9.1. | Pure logic is cheaper to rewrite in C′, where it is verified; I/O and audited specialist code (TLS, compression, database drivers) is not. Wrapping it as a provider reuses the Rust ecosystem while keeping every side effect visible to the effect checker. Direct FFI would bypass that checker. | §2, §3.1, §5.2, §6.4, §8; amends OQ-5 | Phase 3 start | Decided |

### 9.1 OQ-10 detail — provider SDK

**Amends OQ-5.** The four built-in providers remain, but the catalogue is no longer closed. Providers stay statically linked; dynamic loading remains a non-goal.

**Authoring.** A provider is a Rust crate that depends on a new workspace crate, `cprime-provider`. It implements one capability and declares, for each operation, its name, its C′ signature and whether it is `deterministic`. The four built-in providers are rewritten on the SDK, so there is one provider mechanism, not two.

**Types at the boundary.** Restricted in v1 to `Bool`, `U8`, `U64`, `I64`, `Bytes`, `FixedBytes[N]`, `Vec` of these, and `Result[T, E]` over these. Richer C′ types are built on the C′ side from these. This keeps conversion between Rust ownership and C′ reference-counted values to copies of plain data.

**Registration and build.** A project manifest lists the provider crates it uses. `cprimec` then builds a **project-specific runtime** with Cargo, linking those crates statically. `cprimed` loads the same build for JIT checks and fuzzing, so development and release run identical provider code. A rebuild is needed only when the provider list or a provider crate changes. The manifest's name, location and format are defined during `/spec`.

**Binding checks.** `bind <Capability> = <provider>` (§5.2) is accepted only if the provider implements every operation of the `.cprime` capability with an identical signature and the same `deterministic` flag.

**Trust.**

- Every provider joins the trusted computing base (§5.2). Its code is not verified; the verifier models its operations as described in §5.4.
- Each release build reports a **provider inventory**: every linked provider crate and its exact version from `Cargo.lock`, so the trusted code is visible per build.
- The fuzzing harness calls each `deterministic` operation twice with the same inputs and reports any difference as a new diagnostic, `NonDeterministicProvider`. This catches some, not all, false determinism claims.

---

## Appendix A — `.cprime` grammar sketch (EBNF)

Not normative; the normative grammar is in `docs/spec/components/cprime-syntax.md`. `TERM` is the end of a statement: a newline after an identifier, literal, `)`, `]`, `}`, `proved`, `tested` or `variant`, or a written `;`.

```ebnf
file        = "module" path TERM { item } ;
item        = alias | struct | enum | capability | spec | entry | bind ;

alias       = "type" IDENT [ generics ] "=" type_expr TERM ;
struct      = "struct" IDENT [ generics ] "{" { field TERM } "}" ;
enum        = "enum" IDENT [ generics ] "{" variant TERM variant TERM { variant TERM } "}" ;
generics    = "[" IDENT { "," IDENT } "]" ;
type_expr   = path [ "[" type_arg { "," type_arg } "]" ] ;
field       = IDENT { "," IDENT } type_expr ;
variant     = IDENT [ "{" { field TERM } "}" ] ;

capability  = "capability" IDENT "{" { cap_op TERM } "}" ;
cap_op      = [ "deterministic" ] IDENT "(" [ params ] ")" [ type_expr ] ;

spec        = "spec" IDENT "(" [ params ] ")" [ type_expr ] "{" { clause TERM } "}" ;
clause      = "uses" path { "," path }
            | "assurance" ( "proved" | "tested" )
            | "unconstrained" waiver { "," waiver }
            | "requires" expr
            | "ensures" expr ;              (* in this order; requires and ensures may repeat *)
waiver      = IDENT | IDENT "." IDENT | "variant" ;

entry       = "entry" IDENT "(" [ params ] ")" [ type_expr ] "{" [ "uses" path { "," path } TERM ] "}" ;
bind        = "bind" path "=" path TERM ;

params      = IDENT { "," IDENT } type_expr { "," IDENT { "," IDENT } type_expr } ;
path        = IDENT { "::" IDENT } ;
expr        = (* boolean and arithmetic expressions over parameters, `result`,
                 prelude functions and deterministic capability operations;
                 operators: ==> == != < <= > >= + - * / % ++ && || ! is;
                 byte literals x"…" *) ;
```
