<a href="https://www.cprime.cc"><img alt="C prime" src="assets/logo/wordmark-cprime.png"/></a>

### Contract-governed software construction for machine-generated implementations

**A verification-first systems language and compiler in which humans write contracts and AI agents write the implementations.**

> **Status: design stage.** No compiler code exists yet. This repository currently holds the concept, the high-level design (HLD, revision 0.10.0), the approved low-level spec set, and TextMate grammars for `.cprime` and `.opq`. See [Project status](#project-status).

---

## Why C′ exists

AI agents can now produce very large codebases that no human reads line by line. Whether that code can be trusted depends heavily on the language it is written in.

### Rust: the best accidental target

Rust is currently the strongest mainstream choice for unattended AI code generation:

- **The compiler acts as the reviewer.** Ownership, borrowing and a strict type system move whole classes of error (memory corruption, data races) to compile time. An agent in a loop gets a rigorous check without a human reading the output.
- **Diagnostics read like prompts.** `rustc` explains why a lifetime ended and often suggests the fix, which is close to an ideal next-turn prompt for a model.
- **Uniform tooling.** `cargo check`, `cargo test` and `cargo clippy` give the agent a deterministic harness to iterate against.

### Rust: an expensive target

The same strictness that makes the *output* safe makes the *process* costly:

- **Borrow-checker loops.** Lifetimes need global reasoning across a program; models generate locally, token by token. Agents fix one lifetime error and create another elsewhere, and can burn very large token budgets doing so.
- **Appeasement over idiom.** The easy way out of a borrow-checker dead end is `.clone()`, `Arc<Mutex<T>>` or boxing. The tests pass; the program is slower and less idiomatic than hand-written code.
- **Slow feedback.** Compile times stretch every write–compile–read-error cycle, and the loop may run thousands of times.
- **Token-heavy syntax.** Lifetime annotations, turbofish and trait bounds exist for human readers and disambiguation, and every one costs tokens.
- **Tests are not specifications.** An agent optimises for the tests in front of it. Passing a large test suite says nothing about behaviour the suite does not exercise.
- **Unread code has unknown authority.** If nobody reads the implementation, nothing stops it from reading environment variables or opening a socket.

### The C′ bet

If engineering is splitting into *human intent* (specifications) and *machine implementation*, the implementation language should be designed for the machine and the specification language for the human. C′ makes that split explicit:

| Problem with Rust as an AI target | C′'s answer (v1) |
|---|---|
| Borrow-checker loops, lifetime annotations | No borrow checker. Immutable values plus compile-time **Perceus reference counting** with FBIP (in-place) reuse. The agent never annotates lifetimes or ownership. |
| Tests stand in for the specification | Humans write **`requires` / `ensures` contracts**. Implementations are discharged against them with **Z3**; property-based fuzzing is a fallback only where the spec's `assurance` policy allows it. |
| Unread code has ambient authority | An **explicit effect system**: every side effect is an operation of a named capability, and an implementation may perform only the capabilities its spec lists in `uses`. There is no ambient filesystem, network, clock, randomness or environment access. |
| Token-heavy text, imports, files | Agents write compact S-expression `.opq` files with no imports and no restated types. A **content-addressed semantic graph** lets the agent ask for only the context a spec needs (`get_context`). |
| Slow, prose-only feedback | A long-running daemon (`cprimed`) with a warm incremental database returns **structured JSON diagnostics** — including a counterexample whenever the solver finds one — under a tiered latency budget. |

C′ does **not** guarantee anything a contract does not state. A result field that no `ensures` clause mentions is rejected (`UnconstrainedOutput`) unless the spec waives it by name, so an omitted field is visible rather than silent. This is an omission check, not a completeness guarantee: a field mentioned only by a clause that adds no information, such as `result.n >= 0` for a `U64`, passes it.

---

## How it works

### Two surfaces, one semantic graph

**Humans write `.cprime` files** — types, capabilities, contracts and policy:

```text
// auth.cprime
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

**Agents write `.opq` files** — one implementation per spec, bound positionally:

```lisp
;; implements Auth::generate_token
(impl Auth::generate_token [uid now secret]
  (let [exp (+ now 3600)
        msg (++ (u64_be uid) (u64_be exp))
        sig (perform CryptoCap::hmac_sha256 secret msg)]
    (SessionToken uid exp sig)))
```

The `.opq` file does not restate types or effects; they come from the spec, so the two cannot disagree.

### What an accepted implementation guarantees

These are planned guarantees of a compiler that does not exist yet, not demonstrated results. An implementation is accepted only if it:

1. type-checks;
2. performs no effect outside its spec's `uses` list;
3. assuming its spec's `requires`, establishes every `ensures` clause and the unreachability of every trap, and establishes the `requires` of every spec it calls — at the evidence level its `assurance` policy demands (`proved` or `tested`);
4. terminates, unless the spec grants `Diverge`, provided every provider call returns. A terminating program can still trap on a configured CPU-time, memory or call-depth limit.

Arithmetic is checked: overflow, division by zero and out-of-bounds indexing are traps, and the verifier must prove no trap is reachable under `requires`.

The guarantees are conditional. A proof assumes the contracts of the specs an implementation calls and the verifier's model of each provider operation. It carries over to the running program only if those callees and providers meet them and the trusted computing base — `cprimec`/`cprimed`, Z3, Cranelift, LLVM and every provider — is correct. A `requires` that no input satisfies makes every obligation hold trivially; v1 does not yet detect it.

### The agent loop

The daemon exposes three operations over JSON-RPC 2.0 and as MCP tools:

| JSON-RPC method | MCP tool | Purpose |
|---|---|---|
| `get_context` | `cprime_get_context` | Return only the definitions reachable from a spec |
| `check_impl` | `cprime_check_impl` | Fast feedback: parse, type, effect and termination checks, IR lowering, JIT (T0–T1) |
| `submit_impl` | `cprime_submit_impl` | Full verification (T0–T3); on acceptance, write the `.opq` file atomically |

The agent talks to the daemon over stdio, so no network port is opened. Several agent sessions can work on one project at once, each with its own daemon. An accepted write takes a short project lock and is refused with a conflict if another session changed the files that verification relied on; the agent then re-reads and resubmits.

Latency targets (p95, warm database, ≤ 200 IR instructions):

| Tier | Work | Target |
|---|---|---|
| T0 | Parse, resolve, type-check, effect-check, termination check | ≤ 5 ms |
| T1 | IR lowering, Perceus pass, Cranelift JIT | ≤ 20 ms |
| T2 | SMT verification | ≤ 250 ms per obligation; ≤ 2 s per submission |
| T3 | Fuzzing (only under `assurance tested`, when T2 is inconclusive) | 10,000 iterations, ≤ 5 s |

When the solver finds inputs that break a clause, the rejection carries them as a counterexample the agent can act on directly — for example, the inputs that make `result.expires_at` differ from `now + 3600`. Not every rejection has one: a solver timeout or `unknown` result is reported as inconclusive, which is not evidence of a defect in the implementation.

### Reusing the Rust ecosystem

`.opq` has no FFI. Foreign code runs only inside **host providers**: Rust crates built on the `cprime-provider` SDK, statically linked into a project-specific runtime and bound to capabilities in `.cprime` (`bind CryptoCap = host::Crypto`). v1 ships `host::Crypto`, `host::Clock`, `host::Random` and `host::Log`; there are no built-in network or filesystem providers. Every release build reports a provider inventory, so the trusted code is visible per build.

### Servers and multi-threading

A release binary can be a long-running server. A `server` item names a **handler** spec and a **transport**: a provider, written in Rust on the provider SDK, that receives requests and sends responses (C′ ships a sample HTTP transport; none is built in). For each request, the runtime spawns a short-lived, share-nothing task that runs the handler on one of several worker threads, in its own memory:

```text
spec handle(req Request) Response {
    uses Log
    ensures req.path == x"2f6865616c7468" ==> result.status == 200
}

server Api {
    handler Api::handle
    transport acme::HttpTransport
}
```

The handler is an ordinary spec: it is type-checked, effect-checked and verified like any other, must terminate, and cannot assume anything about its untrusted input (no `requires`). A trap or a per-request limit fails only that request; the server keeps serving. No value is ever shared between threads, so reference counts stay non-atomic and programs that are not servers pay nothing for this. Within one program, `par_map` spreads a computation over the cores the same way.

---

## Architecture

The compiler is a Rust (2024 edition) Cargo workspace, planned as:

```text
crates/
├── cprime-core/       # Namespace table, content hashes, IR definitions, Salsa database
├── cprime-syntax/     # Parsers for .cprime and .opq, with error recovery
├── cprime-types/      # Bidirectional type checker
├── cprime-effects/    # Capability/effect inclusion checker, termination check
├── cprime-memory/     # Perceus RC insertion and FBIP reuse
├── cprime-verify/     # Z3 lowering, proof obligations, fuzzing harness
├── cprime-codegen/    # Cranelift JIT (iteration) and LLVM AOT (release)
├── cprime-daemon/     # JSON-RPC 2.0 / MCP server library, source sync, persistence
├── cprime-provider/   # Runtime ABI and provider SDK
└── cprime-cli/        # The cprimec binary: daemon launcher, runtime and release builds, inventory
```

Four built-in provider crates (`cprime-host-crypto`, `-clock`, `-random`, `-log`) sit alongside the workspace. There is no generic `cprimed` binary: each project's runtime build generates its own `cprimed` with its providers statically linked, and `cprimec daemon` launches it, rebuilding first when the providers have changed.

Key dependencies: `salsa`, `chumsky`, `blake3`, `z3`, Cranelift, `inkwell` (LLVM), `arbitrary`, `tokio`, `serde`.

## Scope of v1

Deliberately out of scope: region inference, shared-memory multi-threading (v1 runs `par_map` and per-request server tasks on share-nothing workers; program-spawned tasks with channels come next), mutable references, loop constructs (iteration is recursion with `decreases` measures), FFI from `.opq`, user-defined effect handlers, side-channel resistance, information-flow control (capabilities limit which effects code may perform, not what data reaches them: a permitted `Log` operation can still receive a secret), a human-readable rendering of `.opq`, built-in network and filesystem providers, and dynamically loaded providers.

## Claims still to be measured

The acceptance guarantees, the latency targets and three headline claims are hypotheses, not results. The roadmap measures the three claims and reports the outcome whether or not they hold:

- **H1** — for the same contracts, `.opq` uses at least 40% fewer model tokens than idiomatic Rust, with no drop in first-submission acceptance rate (50-task corpus, same model and prompt for both languages).
- **H2** — the Go-like `.cprime` syntax is at least as easy for people to read and write as the earlier Rust-like syntax (a reading and writing test with 8–10 developers new to C′).
- **Performance** — release builds run within 2× the time of equivalent hand-written Rust on the benchmark suite.

The case for C′ does not rest on H1: verified acceptance of agent-written code stands on its own.

---

## Roadmap

| Phase | Scope |
|---|---|
| 1 — Front end | Parsers, namespace table and hashing, type checker (sum types, exhaustive `match`, monomorphisation), effect and termination checks |
| 2 — IR and memory | Lowering to C′-IR, Perceus RC and FBIP, IR interpreter as reference oracle |
| 3 — JIT and daemon | Cranelift JIT, `cprimed`, MCP tools, persistence, provider SDK and built-in providers, H1 and H2 measurement |
| 4 — Verification | Z3 lowering, proof obligations, `submit_impl`, evidence levels |
| 5 — Fuzzing and release | Fuzzing harness, `assurance tested`, LLVM AOT, server runtime and sample HTTP transport, benchmark suite |

Each phase ends at a testable exit criterion, not a date.

## Project status

| Artefact | Location | State |
|---|---|---|
| Concept | [`docs/cprime-concept.md`](docs/cprime-concept.md) | Draft, revision 4 |
| High-level design | [`docs/design/cprime/`](docs/design/cprime/README.md) | Draft, revision 0.12.0 |
| Low-level spec set | [`docs/spec/`](docs/spec/) | Approved on 2025-12-20, including RFC 05 (Go-like `.cprime` syntax) and RFC 06 (servers) (self-audit: [`phase-6-audit.md`](docs/spec/phase-6-audit.md)) |
| Compiler source | — | Not started |

## Repository layout

```text
docs/
├── cprime-concept.md # Technical and architectural concept
├── design/cprime/    # High-level design: context, architecture, sequences, domain, ADRs, risks
└── spec/             # Low-level design: analysis, architecture, components, generation playbook
```

## Further reading

Start with the [HLD overview](docs/design/cprime/00-overview.md), then the [architecture](docs/design/cprime/02-architecture.md) and [decisions](docs/design/cprime/09-decisions.md). The [concept](docs/cprime-concept.md) contains the full IR definition and the grammar sketch.

## Licence

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
- MIT licence ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in this work by you, as defined in the Apache-2.0 licence, is dual licensed as above, without any additional terms or conditions.
