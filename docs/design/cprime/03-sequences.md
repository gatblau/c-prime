# C′ — Principal Sequences

> **Mode:** draft
> **Generated from:** inline context — `docs/cprime-concept.md` (Draft, revision 2)
> **Revision:** 0.12.0
> **Last updated:** 2025-12-20

## Sequence: implement a spec (golden path)

```mermaid
sequenceDiagram
  autonumber
  participant A as AI agent
  participant D as cprimed
  participant DB as Salsa database
  participant FE as Checker, T0
  participant BE as IR, RC and JIT, T1
  participant V as Verifier, T2
  participant FS as Project files

  A->>D: get_context for a spec name
  D->>DB: resolve name through namespace table
  DB-->>D: reachable types, capability signatures, prelude functions
  D-->>A: context
  loop until fast checks pass
    A->>D: check_impl with spec name and source
    D->>FS: compare file metadata, re-read changed sources (ADR-030)
    D->>FE: parse, resolve, type, effect, termination, completeness
    FE-->>D: typed implementation or diagnostics
    D->>BE: lower, insert RC, JIT compile
    BE-->>D: compiled function or diagnostics
    D-->>A: status, diagnostics, timings
  end
  A->>D: submit_impl with spec name and source
  D->>FE: T0
  D->>BE: T1
  D->>V: obligations per ensures clause, trap site, decreases clause, solved in parallel
  V->>V: vacuity check of the contract first (cached per spec, ADR-052)
  V-->>D: unsat for every obligation, evidence proved
  D->>FS: take write lock, check files unchanged since request start (A-3)
  D->>FS: write temporary file, rename to module dir slash spec.opq
  D->>DB: record accepted implementation
  D->>DB: effective evidence across callees (ADR-053)
  D-->>A: accepted, evidence proved, effective evidence, timings
```

- **Trigger:** an AI agent is asked to implement a named spec.
- **Result:** the implementation is written to `<module-dir>/<spec>.opq` on disk (e.g. `auth/generate_token.opq`), and the reply reports `accepted` with evidence `proved` and the effective evidence across its callees.
- **Error posture:** any T0/T1 failure returns `status: "rejected"` with diagnostics carrying integer node ids, and nothing is written; the agent revises and retries. A protocol fault (unknown method, malformed parameters, daemon failure) returns a JSON-RPC `error` object instead. A failed atomic write means no acceptance is reported (A-14). If another session changed the files this submission relied on, the write is refused with a conflict and the agent re-reads and resubmits (A-3).

## Sequence: submission rejected with a counterexample

```mermaid
sequenceDiagram
  autonumber
  participant A as AI agent
  participant D as cprimed
  participant V as Verifier
  participant Z as Z3

  A->>D: submit_impl
  Note over D: T0 and T1 pass
  D->>V: obligations
  V->>Z: requires and not obligation
  alt model found
    Z-->>V: sat, model
    V->>V: replay counterexample in the interpreter, slice the steps
    V->>V: concrete replay with real deterministic providers and accepted callees (ADR-055)
    V-->>D: ContractViolation or ReachableTrap with counterexample, witness trace and concrete status, or NonDecreasing with counterexample
    D->>D: fill step spans and names from the submission
    D-->>A: rejected, evidence none, diagnostics with inputs, expected, actual and witness
  else unknown, timeout or budget cut-off under assurance proved
    Z-->>V: unknown
    V-->>D: VerificationInconclusive
    D-->>A: rejected, evidence none
  end
```

- **Trigger:** `submit_impl` for an implementation that passes T0/T1 but violates its contract, can reach a trap, or cannot be proved within budget.
- **Result:** rejection with a counterexample (inputs, expected, actual) and, for `ContractViolation` and `ReachableTrap`, a witness trace (ADR-047) and a `concrete` or `abstract` classification (ADR-055); or `VacuousContract` when the contract itself admits no input or no result (ADR-052); or `VerificationInconclusive`; nothing is persisted.
- **Error posture:** fail fast, no silent downgrade from `proved` to `tested` (ADR-025). The agent uses the counterexample to correct the implementation.

## Sequence: fuzzing fallback under `assurance tested`

```mermaid
sequenceDiagram
  autonumber
  participant D as cprimed
  participant V as Verifier
  participant G as Fuzz generator
  participant J as JIT code
  participant P as Providers

  D->>V: T2 obligations
  V-->>D: at least one unknown or timeout
  D->>G: start T3, iteration and time budget from configuration
  loop until budget spent or violation found
    G->>G: generate structured input, keep only if requires holds
    G->>J: call implementation in memory
    J->>P: deterministic operations call real provider, others call stubs
    J-->>G: result or trap
    G->>G: check ensures, record violation
  end
  G->>P: call each deterministic operation twice with same inputs
  G-->>D: tested, or counterexample, or NonDeterministicProvider
```

- **Trigger:** T2 returns `unknown`/timeout for at least one obligation and the spec declares `assurance tested`.
- **Result:** evidence `tested` and acceptance if the configured budget (default 10,000 iterations, ≤ 5 s) finds no violation; otherwise a rejection with a counterexample or `NonDeterministicProvider`.
- **Error posture:** fuzzing never performs real I/O; non-deterministic operations are stubbed. When rejection sampling cannot find inputs satisfying `requires` within budget, the result is `VerificationInconclusive` at T3 with `reason: fuzz_budget_incomplete`.

## Sequence: daemon start-up

```mermaid
sequenceDiagram
  autonumber
  participant U as User or agent host
  participant C as cprimec daemon (launcher)
  participant CG as Cargo
  participant D as project cprimed
  participant FS as Project files
  participant DB as Salsa database

  U->>C: start
  C->>FS: read cprime.toml and runtime fingerprint
  alt fingerprint missing or different
    C->>CG: build project runtime package
    CG-->>C: cprimed executable and runtime library
    C->>FS: write fingerprint
  end
  C->>D: replace launcher with project cprimed, stdio inherited
  D->>FS: read cprime.toml, .cprime and .opq files
  D->>DB: load sources, build namespace table
  D-->>U: ready to serve on stdio
```

- **Trigger:** an agent host starts `cprimec daemon` for a project.
- **Result:** the project's `cprimed` serves JSON-RPC and MCP on stdio.
- **Error posture:** a failed Cargo build stops the launch with the build output on stderr and no fingerprint change. Behaviour when an existing `.opq` no longer satisfies its edited spec: stale (ADR-031). A second daemon on the same project starts normally and serves its own client (A-3).

## Sequence: serve a request (ADR-051)

```mermaid
sequenceDiagram
  participant T as Transport provider
  participant W as Worker thread
  participant H as Handler task (compiled C′)
  W->>T: recv(worker)
  T-->>W: request fields and a reply token
  W->>W: spawn a task: new task heap, copy the request in, set the recovery point
  W->>H: handle(request)
  alt returns
    H-->>W: response
    W->>T: reply(token, response fields)
  else trap or per-task limit
    H--xW: trap entry (mode 3): one stderr line, return to the recovery point
    W->>T: fail(token, kind)
  end
  W->>W: end the task: task heap empty (normal) or freed in full (abnormal)
```

- **Trigger:** the transport has a request ready when a worker asks for one; a busy worker asks for none (back-pressure).
- **Failure:** a trap, `FuelExhausted`, `StackLimit` or `MemoryLimit` inside the handler ends the task, fails that request through the transport, and leaves the worker serving.
- **Shutdown:** on `SIGTERM` or `SIGINT` the transport stops; `recv` returns `Shutdown`; workers finish their task and exit; the process exits 0, or 8 if `limits.shutdown_grace_ms` passes first.

## Sequence: project runtime build and release

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant C as cprimec
  participant FS as Project files
  participant CG as Cargo
  participant L as LLVM

  U->>C: build release
  C->>FS: read manifest and sources
  alt provider list or a provider crate changed
    C->>CG: build project-specific runtime, static linking
    CG-->>C: runtime and Cargo.lock
  end
  C->>L: AOT compile current implementations and entry point
  L-->>C: object file
  C->>CG: build release package: generated main, object file, runtime library
  CG-->>C: release binary
  C-->>U: release binary and provider inventory
```

- **Trigger:** a user runs `cprimec build --release`.
- **Result:** a native release binary and a provider inventory listing every linked provider crate with its exact `Cargo.lock` version.
- **Error posture:** a failing Cargo build or an invalid `bind` stops the build with a report; no partial binary is emitted (A-15).
