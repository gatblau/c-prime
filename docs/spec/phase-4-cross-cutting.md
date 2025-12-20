# Phase 4 — Cross-Cutting Concern Specifications

> **Spec set:** cprime
> **Mode:** greenfield
> **derivedFromHld:** 0.12.0
> **HLD-impact status:** adjudicated — see `phase-1-analysis.md` banner.
> **Status:** Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> **Last updated:** 2025-12-20

## Contents

- [Concerns out of scope in v1](#concerns-out-of-scope-in-v1)
- [SPEC: Authorisation (capability model)](#spec-authorisation-capability-model)
- [SPEC: Errors and Exit Codes](#spec-errors-and-exit-codes)
- [SPEC: Logging](#spec-logging)
- [SPEC: Metrics (tier timings)](#spec-metrics-tier-timings)
- [SPEC: Configuration](#spec-configuration)
- [SPEC: Input Validation](#spec-input-validation)
- [SPEC: Graceful Shutdown](#spec-graceful-shutdown)
- [SPEC: Testing and Measurement](#spec-testing-and-measurement)

## Concerns out of scope in v1

Recorded as SA-33 in the Assumptions Register. Each is `N/A` for the reason given; no component implements it.

| Concern | Decision | Reason |
|---|---|---|
| Authentication | N/A | `cprimed` serves only its parent over stdio (ADR-029 proposed, A-1). |
| Tracing | Split | Tracing of `cprimed` itself: N/A (one executable, direct function calls; per-tier timings cover the need, HLD `07-cross-cutting.md`). Traces of programs: specified by D22–D28 (ADR-047 witness traces, ADR-048 release trap locations); see Errors and Exit Codes, Logging, Metrics and Testing and Measurement below. |
| Rate limiting | N/A | One client per `cprimed`; work per request is bounded by T2/T3 budgets and fuel. |
| Pagination | N/A | `get_context` returns one spec's reachable definitions; no list endpoints. |
| CORS | N/A | No HTTP interface. |
| Health checks | N/A | Not a network service; a failed stdio call is the signal. `ping` (MCP) answers `{}`. |
| Database migrations | N/A | No database. On-disk format versioning is the hash domain tag `cprime.def.v1` and the header version `v1` (SA-28); a change to either is a new tag, and old `.opq` files then report `Stale`. |

---

### SPEC: Authorisation (capability model)
**File:** `crates/cprime-effects/src/effects.rs`, `crates/cprime-effects/src/bindings.rs`, `crates/cprime-types/src/check.rs`, `crates/cprime-codegen/src/{jit,aot,interp}.rs` | **Package:** cross-cutting | **Phase:** 1 (checks), 3 (enforcement in back ends) | **Dependencies:** `cprime-effects`, `cprime-types`, `cprime-codegen`, `cprime-provider`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose
Guarantees that compiled programs have no ambient authority: every side effect is an operation of a capability listed in the spec's `uses`, foreign code runs only inside statically linked providers, and only `.cprime` can bind a capability to a provider. There is no RPC-level authorisation (A-1).

#### Approach
The check is static and set-based (ADR-005): the transitive effect set must be a subset of `uses`. Enforcement in back ends is structural — the IR has no instruction that reaches foreign code except `Perform`, and `Perform` targets come only from bindings. Rejected: run-time capability tokens (two models instead of one, non-goal); a sandbox around JIT code (does not stop a provider, which is trusted code anyway).

#### Shared Context
```rust
pub struct CapId(pub u32);                 // CapId(0) = Diverge
pub enum OpTarget { Provider { flat_index: u32 }, Stub }
pub struct BindingCheck { pub bindings: BTreeMap<CapId, u32>, pub diagnostics: Vec<Diagnostic> }
// TSpec.uses: BTreeSet<CapId>; TExprKind::Perform { cap, op, deterministic, args }; TExprKind::Call { func, args }
```
Diagnostic kinds used: `UnauthorizedEffect`, `TypeMismatch`, `UnknownName`.

#### Public Interface
No interface of its own. Rules enforced by: `cprime_effects::check_effects` (subset rule), `cprime_effects::check_bindings` (binding validity), `cprime_types::check_spec` (contracts may use only `deterministic` operations), `cprime_codegen` (a `Perform` with `OpTarget::Stub` is `CodegenError::UnboundOperation` in AOT; in the JIT it calls the stub slot; in the interpreter it traps `Unreachable`).

##### Example
`spec s() { uses Log }` with an implementation that performs `CryptoCap::sha256` → `UnauthorizedEffect`, message ``effect CryptoCap is not in `uses Log` ``.

#### Internal Logic
1. `.opq` grammar has no form for foreign calls, installing effect implementations, or binding (cprime-syntax); attempts are `ParseError`.
2. `check_effects` computes own `Perform` capabilities ∪ callees' declared `uses`; any capability outside `spec.uses` → one `UnauthorizedEffect` per capability at the first offending node.
3. `check_bindings` accepts `bind` only in the entry's module and only when the provider implements every operation with identical signature and `deterministic` flag.
4. `cprime-daemon` builds `Program.op_targets` from bindings only; unbound operations are `Stub`.
5. Fuzzing calls a real provider only for bound `deterministic` operations (D4, SA-30); every other operation is a stub, so fuzzing performs no real I/O.
6. Release builds refuse a program with any unbound capability in the entry's transitive `uses` (`check_bindings` diagnostics are fatal in `cprime-cli`).

#### Data Model
N/A — no persistent state.

#### Error Table
| Condition | Status | Code | Response Body |
|---|---|---|---|
| Effect outside `uses` | Diagnostic (rejected) | `UnauthorizedEffect` | `{"kind":"UnauthorizedEffect","tier":"t0","severity":"error","node":7,"message":"effect CryptoCap is not in `uses Log`"}` |
| Provider signature differs from capability | Diagnostic (release fatal) | `TypeMismatch` | `message: "provider host::Crypto operation sha256 returns FixedBytes[32], capability declares Bytes"` |
| `bind` outside the entry module | Diagnostic (release fatal) | `TypeMismatch` | `message: "bind is allowed only in module Main, which declares the entry"` |
| Non-deterministic operation in a contract | Diagnostic (rejected) | `TypeMismatch` | `message: "operation Clock::now_unix_seconds is not deterministic and cannot appear in a contract"` |

#### Acceptance Criteria (Gherkin)
Feature: Authorisation (capability model)

  Scenario: Happy path — declared capability
    Given spec `Auth::generate_token` with `uses CryptoCap`
    When its implementation performs only `CryptoCap::hmac_sha256`
    Then no `UnauthorizedEffect` diagnostic is produced

  Scenario: Edge case — effect reached through a callee
    Given spec `f` with no `uses` line calls spec `g` with `uses Log`
    When `f` is checked
    Then `UnauthorizedEffect` names `Log` at the node of the call to `g`

  Scenario: Error path — fuzzing never performs real I/O
    Given a bound non-deterministic operation `Log::info`
    When T3 runs
    Then the JIT slot for `Log::info` holds a stub and nothing is written to stderr by the program

  Scenario: Error path — unbound capability in a release build
    Given an entry with `uses CryptoCap` and no `bind`
    When `cprimec build --release` runs
    Then the exit code is 1 and stderr names the missing bind

#### Performance, Security, Observability
- **Performance targets:** effect check ≤ 0.5 ms p95 within T0's 5 ms.
- **Security:** the trusted computing base is `cprimec`/`cprimed`, Z3, Cranelift, LLVM and every provider; the provider inventory makes the provider part visible per release.
- **Observability:** `UnauthorizedEffect` diagnostics appear in replies; no separate log line.

#### Gaps
None.

---

### SPEC: Errors and Exit Codes
**File:** `crates/cprime-core/src/diag.rs`, `crates/cprime-daemon/src/rpc.rs`, `crates/cprime-cli/src/main.rs` | **Package:** cross-cutting | **Phase:** 1–5 | **Dependencies:** `cprime-core`, `cprime-daemon`, `cprime-cli`, `cprime-provider`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose
One registry of every failure signal C′ emits — diagnostics, JSON-RPC errors, exit codes — so clients branch on stable codes, and compile or verification failures never look like protocol faults (ADR-010).

#### Approach
Three channels, each with a closed code set: diagnostics inside `result` for anything the agent can fix; JSON-RPC `error` for faults the agent cannot fix by editing code; exit codes for `cprimed` start-up, `cprimec`, and release binaries. Library crates return typed errors (`ConfigError`, `IrVerifyError`, `CodegenError`, `VerifyError`, `OpenError`) and never panic on user input; only `marshal` panics on a caller bug (SA-42). Rejected: one error code space for all channels — exit codes and JSON-RPC codes have fixed numeric ranges.

#### Shared Context
`Diagnostic`, `DiagnosticKind`, `Tier`, `Severity`, `Span`, `Counterexample` — Phase 2 §2C (copied in `components/cprime-daemon.md`). `reason` values: `solver_unknown`, `timeout`, `budget_cut_off`, `fuzz_budget_incomplete`, `fuzz_resource_limit`, `callee_unimplemented`, `unsupported_construct` (D44), `provider_failure` (D42), `provider_timeout` (D41). `ConcreteCheck` and its reason values (D43): Phase 2 §2C.

#### Public Interface
**Diagnostic kinds** (severity `error`, except `VacuityUnknown`, whose severity is `note` and which never affects acceptance; SA-32, D37):

| Kind | Tier | Owner |
|---|---|---|
| `ParseError` | T0 | `cprime-syntax` |
| `UnknownName` | T0 | `cprime-core`, `cprime-types`, `cprime-effects` |
| `ArityMismatch`, `TypeMismatch`, `NonExhaustiveMatch`, `UnconstrainedOutput` | T0 | `cprime-types` (`TypeMismatch` also `cprime-effects`, `cprime-daemon`) |
| `UnauthorizedEffect`, `MissingDecreases` | T0 | `cprime-effects` |
| `InvalidParMap` | T0 | `cprime-types`, `cprime-effects` (D15, D16) |
| `ContractViolation`, `ReachableTrap` | T2, T3 | `cprime-verify` |
| `NonDecreasing` | T2 | `cprime-verify` |
| `VerificationInconclusive` | T2, T3 | `cprime-verify` |
| `NonDeterministicProvider` | T3 | `cprime-verify` |
| `InvalidServer` | T0 | `cprime-effects` (D32, ADR-051) |
| `VacuousContract` | T2 | `cprime-verify` (D37, ADR-052) |
| `VacuityUnknown` (severity `note`) | T2 | `cprime-verify` (D37, ADR-052) |

**JSON-RPC errors** (`cprime-daemon`):

| Code | Message | `data` |
|---|---|---|
| -32700 | `parse error` | `{"detail": string}` |
| -32600 | `invalid request` | `{"detail": string}` |
| -32601 | `method not found` | `{"method": string}` |
| -32602 | `invalid params` | `{"detail": string}` |
| -32603 | `internal error` | `{"detail": string}` |
| -32001 | `daemon failure` | `{"reason": "ir_verifier" \| "codegen" \| "solver_init" \| "verifier_failure" \| "unsupported_construct" \| "write_failed" \| "io" \| "internal", "detail": string}` (`unsupported_construct`: D44) |
| -32002 | `unknown spec` | `{"spec": string}` (`get_context` only) |
| -32003 | `module has errors` | `{"file": string, "diagnostics": [Diagnostic]}` |
| -32004 | `restart required` | `{"detail": string}` |
| -32005 | `implementation changed on disk` | `{"impl_file": string}` (`submit_impl` only, SA-05) |

**Exit codes:** `cprimed` — 0 normal end, 1 `--check-providers` found a failing known-answer vector (D40), 2 usage or configuration, 3 `.cprime/` cannot be created, 4 start-up failure. `cprimec` — 0 ok, 1 project check failed (including a runtime refused by a known-answer vector), 2 usage or configuration, 3 environment, 4 toolchain, 5 internal. Release binaries — Phase 2 §2C `ExitCode` (0, 2–9).

**Call outcomes and release exit codes** (RFC 07 criterion 11, ADR-024 as amended, SA-77). A call to a spec ends in exactly one outcome. Termination holds for every spec without `Diverge`, provided every provider call returns.

| Call outcome | `RtTrap` | Release exit code |
|---|---|---|
| Returns | `None` | 0 |
| Traps | `Overflow`, `DivisionByZero`, `IndexOutOfBounds`, `Unreachable` | 3 |
| Exceeds a resource limit: CPU time (fuel in `cprimed`) | `FuelExhausted`; `SIGXCPU` in release | 4 |
| Exceeds a resource limit: memory | `MemoryLimit` | 5 |
| Exceeds a resource limit: call depth | `StackLimit` | 7 |
| Provider fails | `ProviderPanic` | 6 |
| Diverges (only with `Diverge`) | — | none: the program runs until stopped |

Exit code 2 (input does not decode) happens before the entry is called, and exit codes 8 and 9 belong to the server process, not to one call (ADR-051).

##### Example
A T0 failure: `{"jsonrpc":"2.0","id":3,"result":{"status":"rejected","timings_us":{"sync":310,"t0":1450},"diagnostics":[{"kind":"NonExhaustiveMatch","tier":"t0","severity":"error","node":5,"message":"match on Result is missing variant Err"}]}}`. An IR verifier failure: `{"jsonrpc":"2.0","id":4,"error":{"code":-32001,"message":"daemon failure","data":{"reason":"ir_verifier","detail":"dominance: v17 used before its definition in block 3"}}}`.

#### Internal Logic
1. Every library returns diagnostics in a `Vec<Diagnostic>` or a typed error; none writes to stdout.
2. `cprime-daemon` places diagnostics in `result.diagnostics`, ordered by tier, then `span` (file, line, column), then `node`; identical diagnostics (same kind, node, message, clause) are emitted once.
3. Typed internal errors map to `-32001` with the `reason` above; a worker panic maps to `-32603` and the database is rebuilt before the next request.
4. A rejection never carries a JSON-RPC `error`; a protocol fault never carries `result`.
5. Inside compiled programs there are no exceptions; recoverable failure is `Result[T, E]`; traps abort with the release exit codes. Inside `par_map` the lowest-index trap is reported; when a resource limit is hit, which element hits it may vary between runs (SA-54). Release exit codes are unchanged.
6. Release stderr lines (ADR-048, D27): a trap (exit 3) appends ` at <file>:<line>:<column> in <spec>` to `cprime: trap <Kind>`; a provider panic (exit 6) reads `cprime: provider panic in <Cap>::<op> at <file>:<line>:<column> in <spec>`; the call-depth limit (exit 7) reads `cprime: call depth limit exceeded in <spec> (<file>:<line>:<column>)`; inside `par_map` the first line ends ` (par_map element <i>)`. Exits 4 and 5 keep their existing line. Exit codes and the leading text of every line are unchanged, with the prefix `cprime:` (ADR-049), so scripts matching on them keep working. A build with `release.call_chain = true` follows the first line of exits 3–7 with the call chain (D28). No line contains a program value (SA-61).
7. A `ContractViolation` or `ReachableTrap` diagnostic with a counterexample carries a `witness` member when `verify.witness_max_steps > 0` (D22, SA-57); a `not_reproduced` witness never changes the rejection (ADR-025). It also carries `concrete` whatever `verify.witness_max_steps` is (D43, ADR-055); an `abstract` counterexample still rejects.
8. A `note` diagnostic (`VacuityUnknown`, D37) is placed after every `error` diagnostic and may appear in an accepted reply; clients treat `status`, not the presence of diagnostics, as the outcome.

#### Data Model
N/A — no persistent state.

#### Error Table
| Condition | Status | Code | Response Body |
|---|---|---|---|
| Implementation fails a check | `result`, `status: "rejected"` | DiagnosticKind | diagnostics list as in the Example |
| IR verifier failure (compiler bug) | JSON-RPC error | -32001 `ir_verifier` | as in the Example |
| Unknown method | JSON-RPC error | -32601 | `{"code":-32601,"message":"method not found","data":{"method":"compile"}}` |
| Release binary trap | exit code | 3 | stderr `cprime: trap Overflow at auth/generate_token.opq:3:13 in Auth::generate_token` |
| Release binary provider panic | exit code | 6 | stderr `cprime: provider panic in Auth::CryptoCap::hmac_sha256 at auth/generate_token.opq:4:13 in Auth::generate_token` |
| Release binary call-depth limit | exit code | 7 | stderr `cprime: call depth limit exceeded in Math::count (math/count.opq:2:1)` |

#### Acceptance Criteria (Gherkin)
Feature: Errors and exit codes

  Scenario: Happy path — rejection is a result
    Given an implementation with a non-exhaustive match
    When `check_impl` runs
    Then the reply has `result.status` "rejected" and no `error` member

  Scenario: Edge case — duplicate diagnostics are emitted once
    Given two identical `UnknownName` findings on node 4
    When the reply is built
    Then `diagnostics` contains one entry for node 4

  Scenario: Error path — compiler bug is a protocol fault
    Given lowering produces IR that fails the verifier
    When `check_impl` runs
    Then the reply is a JSON-RPC error with code -32001 and reason "ir_verifier"

  Scenario: Error path — release trap names its location
    Given a release binary of the revision-1 contract (no bound on now) with the body (+ now 3600)
    When it runs with now 18446744073709551615
    Then it exits with code 3
    And stderr starts with "cprime: trap Overflow at auth/generate_token.opq:3:13 in Auth::generate_token"
    And stderr contains no input value

#### Performance, Security, Observability
- **Performance targets:** diagnostic serialisation ≤ 0.2 ms for 100 diagnostics.
- **Security:** error `detail` strings contain no submitted source text and no counterexample values (SA-26).
- **Observability:** every `-32001` and `-32603` is logged at `error` with `request_id` and `reason`.

#### Gaps
None.

---

### SPEC: Logging
**File:** `crates/cprime-daemon/src/log.rs` (shared by `cprime-cli` through `cprime-daemon`) | **Package:** cross-cutting | **Phase:** 3 | **Dependencies:** `serde_json`, `cprime-core::config::LogLevel`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose
Structured, local, privacy-conservative logs for `cprimed` and `cprimec` on stderr (A-4, SA-26). stdout of `cprimed` is reserved for JSON-RPC replies.

#### Approach
A small JSON-lines writer in `cprime-daemon`, no logging framework: the fields are few and fixed, and a framework would add a dependency for no gain. Rejected: plain-text logs (not machine-readable); logging to a file (needs rotation; stderr is captured by the agent host).

#### Shared Context
```rust
#[derive(Clone, Copy, serde::Deserialize)] #[serde(rename_all = "lowercase")]
pub enum LogLevel { Error, Warn, Info, Debug, Trace }
// Configuration key: log.level (default "info"); flag --log-level overrides it.
```

#### Public Interface
```rust
pub fn init(level: LogLevel);
pub fn log(level: LogLevel, target: &'static str, msg: &str, fields: &[(&'static str, serde_json::Value)]);
```
Line format — one JSON object, keys in this order, then `\n`:

| Field | Type | Presence |
|---|---|---|
| `ts` | string, RFC 3339 UTC with milliseconds | always |
| `level` | `error` \| `warn` \| `info` \| `debug` \| `trace` | always |
| `target` | `cprimed::start`, `cprimed::rpc`, `cprimed::sync`, `cprimed::pipeline`, `cprimed::persist`, `cprimec::launcher`, `cprimec::runtime`, `cprimec::release` | always |
| `msg` | string | always |
| `request_id` | JSON-RPC `id` | request-scoped lines |
| `method`, `spec`, `status`, `evidence`, `path` | string | when known |
| `timings_us` | object | request completion line |
| `diagnostics` | integer (count only) | request completion line |
| `source`, `counterexample`, `witness` | string, object, object | `trace` only |
| `obligation`, `kind` | integer, string | `verify.witness_mismatch` line |

##### Example
`{"ts":"2025-12-20T10:15:02.114Z","level":"info","target":"cprimed::rpc","msg":"request done","request_id":7,"method":"submit_impl","spec":"Auth::generate_token","status":"rejected","timings_us":{"sync":290,"t0":1900,"t1":8400,"t2":41000},"diagnostics":1}`

#### Internal Logic
1. `init` stores the level once; a line below it is dropped before formatting.
2. `log` serialises the fields with `serde_json`, writes the line to stderr with one `write_all`, and ignores write errors (a broken stderr must not stop request execution).
3. `source`, `counterexample` and `witness` fields (witness step values, D22) are accepted only at `trace`; at any other level they are removed before writing.
3a. `cprime-verify` writes `verify.witness_mismatch` at `error` for every `not_reproduced` witness trace, with `obligation` and `kind` and no step values (ADR-047).
4. Inside compiled programs, logging is the `host::Log` capability (target `program`, SA-44).

#### Data Model
N/A — no persistent state.

#### Error Table
| Condition | Status | Code | Response Body |
|---|---|---|---|
| stderr closed or full | dropped line | — | nothing written; request continues |
| Unknown `log.level` value | start-up failure | `ConfigError::Invalid` | `invalid value for log.level: expected one of error, warn, info, debug, trace` |

#### Acceptance Criteria (Gherkin)
Feature: Logging

  Scenario: Happy path — request completion line
    Given `log.level` is "info"
    When a `check_impl` request completes
    Then stderr receives one JSON line with target "cprimed::rpc", the request id and `timings_us`

  Scenario: Edge case — source omitted below trace
    Given `log.level` is "debug"
    When a submission is logged
    Then the line has no `source` field

  Scenario: Error path — witness mismatch logged without values
    Given `log.level` is "info" and a replay ends `not_reproduced`
    When the rejection is built
    Then stderr receives one `error` line `verify.witness_mismatch` with `obligation` and `kind`
    And the line has no `witness` field

  Scenario: Error path — invalid level
    Given `cprime.toml` sets `log.level = "verbose"`
    When `cprimed` starts
    Then it exits with code 2 and the configuration error message

#### Performance, Security, Observability
- **Performance targets:** ≤ 20 µs per line at `info`.
- **Security:** no submitted source, no counterexample values, no witness step values, no file contents below `trace`; nothing leaves the machine (A-8).
- **Observability:** this spec is the observability surface together with `timings_us`.

#### Gaps
None.

---

### SPEC: Metrics (tier timings)
**File:** `crates/cprime-daemon/src/pipeline.rs` | **Package:** cross-cutting | **Phase:** 3, 4 | **Dependencies:** `std::time::Instant`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose
The only metrics surface in v1 (A-9): every `check_impl` and `submit_impl` reply carries per-tier wall-clock durations so agents and benchmarks can check the latency budget (HLD G5).

#### Approach
Timings travel with each reply; there is no endpoint and no aggregation. Rejected: a Prometheus endpoint (no network listener exists, ADR-029 proposed).

#### Shared Context
Budgets (HLD G5, p95, ≤ 200 IR instructions, warm database): T0 ≤ 5 ms, T1 ≤ 20 ms, T2 ≤ 250 ms per obligation and ≤ 2 s per submission, T3 10,000 iterations ≤ 5 s.

#### Public Interface
`timings_us`: object with integer microsecond members `sync` and `t0` always; `t1`, `t2`, `t3` only when that tier ran; `witness` only when at least one witness replay ran (`VerificationResult.witness_us` is `Some`, D25).

##### Example
`"timings_us": {"sync": 310, "t0": 1900, "t1": 8400, "t2": 41000}`; a counterexample rejection with one witness trace: `"timings_us": {"sync": 310, "t0": 1900, "t1": 8400, "t2": 41000, "witness": 900}`

#### Internal Logic
1. `sync` measures the source scan (SA-06); it is outside the T0 budget.
2. `t0` spans parse through termination check; `t1` spans lowering through JIT compilation; `t2` and `t3` come from `VerificationResult`.
2a. `witness` comes from `VerificationResult.witness_us`: the summed duration of every witness replay and slice and of every concrete replay (D43). It is not part of `t2` or `t3`, so the T2 and T3 budgets are unaffected; its target is ≤ 20 ms p95 per trace on the corpus, measured, not enforced (ADR-047).
3. Durations use `Instant` and are truncated to whole microseconds.

#### Data Model
N/A — no persistent state.

#### Error Table
| Condition | Status | Code | Response Body |
|---|---|---|---|
| T0 rejects | partial timings | — | `{"sync": 300, "t0": 1200}` (no `t1`) |
| JSON-RPC error reply | no timings | — | `error` object only |
| `verify.witness_max_steps = 0` | no `witness` member | — | `{"sync": 300, "t0": 1200, "t1": 8000, "t2": 40000}` on a counterexample rejection |

#### Acceptance Criteria (Gherkin)
Feature: Tier timings

  Scenario: Happy path — accepted submission
    Given a submission accepted with evidence "proved"
    Then `timings_us` has members sync, t0, t1 and t2 and no t3

  Scenario: Edge case — fuzzing ran
    Given a submission under `assurance tested` with an unknown obligation
    Then `timings_us` has a t3 member

  Scenario: Error path — T0 rejection
    Given an implementation with a parse error
    Then `timings_us` has only sync and t0

  Scenario: Edge case — witness timing on a counterexample rejection
    Given a submission rejected with a ContractViolation counterexample and `verify.witness_max_steps` 64
    Then `timings_us` has a witness member
    And an accepted submission's `timings_us` has none

#### Performance, Security, Observability
- **Performance targets:** measurement overhead ≤ 1 µs per tier.
- **Security:** timings reveal nothing beyond durations.
- **Observability:** the request completion log line repeats `timings_us`.

#### Gaps
None.

---

### SPEC: Configuration
**File:** `crates/cprime-core/src/config.rs` | **Package:** cross-cutting | **Phase:** 1 | **Dependencies:** `toml`, `serde`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose
One configuration source per project — `cprime.toml` plus `--config key=value` flags — read once at start (A-10), validated before any work, with nothing operational in `.cprime` (ADR-016).

#### Approach
TOML file with typed sections and `deny_unknown_fields`; flags override; no environment variables (SA-10). Rejected: environment variables (hidden state in agent hosts); hot reload (A-10; a change requires a restart, signalled by `-32004`).

#### Shared Context
Every key, type, default, range and owner: Phase 2 §2D (24 keys, including `limits.max_call_depth`, `limits.workers` (D20), `verify.witness_max_steps` and `release.call_chain` (D29), `limits.request_fuel` and `limits.shutdown_grace_ms` (D36), and `verify.provider_wait_ms` (D41)). Sections: `[project]`, `[[providers]]`, `[limits]`, `[verify]`, `[fuzz]`, `[log]`, `[runtime]`, `[release]`. `ProjectConfig`, `ConfigError` and `load_config`: Phase 2 §2C.

#### Public Interface
`cprime_core::config::load_config(root: &Path, overrides: &[(String, String)]) -> Result<ProjectConfig, ConfigError>`.

##### Example
`cprimec daemon --project . --config verify.submission_budget_ms=4000 --config fuzz.iterations=20000`

#### Internal Logic
1. Read `<root>/cprime.toml` (≤ 1 MiB); absent → `ConfigError::NotFound`.
2. Parse TOML; unknown keys or wrong types → `ConfigError::Toml` with line and column.
3. Apply each override in order; the key must be in §2D and must not be `project.name` or a `providers[]` key → else `UnknownOverride`; the value is parsed as the key's type.
4. Validate ranges and cross-key rules (`submission_budget_ms ≥ obligation_timeout_ms`; exactly one of `version`/`path` per provider; unique provider paths) → `ConfigError::Invalid`.
5. `cprimed` records the file's (modification time, size); a later change makes every C′ method reply `-32004`.

#### Data Model
N/A — the file format is shown in Phase 2 §2C.

#### Error Table
| Condition | Status | Code | Response Body |
|---|---|---|---|
| `cprime.toml` missing | start-up failure, exit 2 | `ConfigError::NotFound` | `cprime.toml not found in /work/auth_service` |
| Value out of range | start-up failure, exit 2 | `ConfigError::Invalid` | `invalid value for fuzz.iterations: must be between 1 and 10000000` |
| Override of an unknown key | start-up failure, exit 2 | `ConfigError::UnknownOverride` | `unknown configuration key in --config: verify.timeout` |
| `verify.witness_max_steps` out of range | start-up failure, exit 2 | `ConfigError::Invalid` | `invalid value for verify.witness_max_steps: must be between 0 and 1024` |
| `release.call_chain` not a boolean | start-up failure, exit 2 | `ConfigError::Toml` (file) or `ConfigError::Invalid` (override) | `invalid value for release.call_chain: expected true or false` |
| File changed after start | JSON-RPC error | -32004 | `restart required` |

#### Acceptance Criteria (Gherkin)
Feature: Configuration

  Scenario: Happy path — defaults apply
    Given `cprime.toml` contains only `[project] name = "auth_service"`
    When configuration loads
    Then `verify.obligation_timeout_ms` is 250 and `limits.max_call_depth` is 100000
    And `verify.witness_max_steps` is 64 and `release.call_chain` is false

  Scenario: Edge case — call chain enabled for one build
    Given `cprime.toml` has no `[release]` section
    And the flag `--config release.call_chain=true`
    Then the effective `release.call_chain` is true

  Scenario: Edge case — flag overrides file
    Given `cprime.toml` sets `verify.submission_budget_ms = 3000`
    And the flag `--config verify.submission_budget_ms=4000`
    Then the effective value is 4000

  Scenario: Error path — inconsistent budgets
    Given `verify.obligation_timeout_ms = 3000` and `verify.submission_budget_ms = 2000`
    Then loading fails with `ConfigError::Invalid` naming `verify.submission_budget_ms`

#### Performance, Security, Observability
- **Performance targets:** `load_config` ≤ 5 ms.
- **Security:** the file holds no secrets; provider `path` values are resolved inside the project root or given as written (Cargo resolves them).
- **Observability:** effective configuration is logged once at `debug` on start.

#### Gaps
None.

---

### SPEC: Input Validation
**File:** `crates/cprime-daemon/src/rpc.rs`, `crates/cprime-syntax/src/lib.rs`, `crates/cprime-core/src/value.rs` | **Package:** cross-cutting | **Phase:** 1, 3 | **Dependencies:** `cprime-daemon`, `cprime-syntax`, `cprime-core`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose
Every byte that enters C′ is bounded and checked before any execution: JSON-RPC envelopes, submitted source, project files, release-binary input, and fuzzed inputs.

#### Approach
Validate at each boundary with fixed limits and fail closed; nothing is executed before T0 passes. Rejected: a schema-validation library (the shapes are small and fixed).

#### Shared Context
Limits: JSON-RPC line ≤ 16 MiB (SA-45); source ≤ 1 MiB, nesting ≤ 256, ≤ 100 diagnostics (SA-36); `cprime.toml` ≤ 1 MiB; JSON value nesting ≤ 128 (`from_json`); `spec` matches `^[A-Za-z_][A-Za-z0-9_]*(::[A-Za-z_][A-Za-z0-9_]*)+$`.

#### Public Interface
No interface of its own; the boundaries are `cprime-daemon` request validation, `cprime_syntax::parse_cprime`/`parse_opq`, `cprime_core::value::from_json`, the generated release `main`, and the `requires` filter in fuzzing.

##### Example
`{"jsonrpc":"2.0","id":9,"method":"check_impl","params":{"spec":"generate_token","source":"(impl …)"}}` → `-32602` with `{"detail":"spec must be a qualified name such as Module::name"}`.

#### Internal Logic
1. `cprime-daemon` validates the envelope, then params: required members present, no extra members, types match, `spec` matches the pattern; failures are JSON-RPC errors (`-32600`, `-32602`).
2. Source text passes through `cprime-syntax` limits; failures are `ParseError` rejections.
3. Release binaries decode stdin with `from_json` against the entry parameter types; failure → exit 2 with `cprime: invalid input at <path>: expected <type>`.
4. Fuzzed inputs are kept only if every `requires` clause is defined and true (SA-19).

#### Data Model
N/A.

#### Error Table
| Condition | Status | Code | Response Body |
|---|---|---|---|
| Unqualified `spec` | JSON-RPC error | -32602 | as in the Example |
| Source over 1 MiB | Diagnostic (rejected) | `ParseError` | `message: "source exceeds 1048576 bytes"` |
| Release input missing a parameter | exit code 2 | — | `cprime: invalid input at $.secret: expected Bytes` |

#### Acceptance Criteria (Gherkin)
Feature: Input validation

  Scenario: Happy path — well-formed request
    Given a `check_impl` request with a qualified spec and a source of 2 KiB
    Then it reaches T0

  Scenario: Edge case — extra params member
    Given a request whose params include `"mode": "fast"`
    Then the reply is -32602 naming `mode`

  Scenario: Error path — oversized line
    Given a request line of 17 MiB
    Then the reply is -32600 with `id: null` and the next line is still read

#### Performance, Security, Observability
- **Performance targets:** envelope validation ≤ 50 µs for a 64 KiB line.
- **Security:** bounded memory per request; nothing runs before T0 and T1 pass; fuzzing performs no real I/O.
- **Observability:** validation failures are logged at `warn` with `request_id` and the error code.

#### Gaps
None.

---

### SPEC: Graceful Shutdown
**File:** `crates/cprime-daemon/src/server.rs`, `crates/cprime-daemon/src/persist.rs` | **Package:** cross-cutting | **Phase:** 3 | **Dependencies:** `tokio`, `libc`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose
A crash, signal or closed stdin never leaves a partial `.opq` and never reports an acceptance that is not on disk (A-14, SA-27).

#### Approach
Temporary file + `fsync` + `rename` + directory `fsync` before the reply; signals abandon the request in progress. Rejected: draining the in-flight request on `SIGTERM` (T2 + T3 can take 7 s by default; agent hosts kill sooner).

#### Shared Context
SA-27: end of stdin → finish the current request, reply, exit 0. SA-05: the project write lock is held only while an accepted file is written. `SIGINT`/`SIGTERM` → abandon the request without replying, exit 0 within 200 ms.

#### Public Interface
No interface of its own; behaviour of `cprime_daemon::run`.

##### Example
`SIGTERM` during T2 of `submit_impl` → no reply, no write, exit 0 in ≤ 200 ms; the `.opq` on disk is unchanged.

#### Internal Logic
1. Take the project write lock (SA-05) and run the conflict check (`cprime-daemon` §H); a conflict replies `-32005` and writes nothing. Write the accepted source (header + body) to `.<spec>.opq.tmp-<pid>` in the target directory, `fsync` it, `rename` over `<spec>.opq`, `fsync` the directory, then release the lock and reply.
2. A write failure removes the temporary file and replies `-32001` reason `write_failed`; no acceptance is reported.
3. At start-up, while holding the project write lock, `cprimed` deletes leftover `.*.opq.tmp-*` files in the project tree (logged at `warn`). Holding the lock guarantees no other `cprimed` is mid-write, so no live temporary file is removed.
4. The operating system releases the project write lock when `cprimed` exits, including after a crash or a signal during a write; the temporary file left behind is removed by step 3 on a later start.

#### Data Model
On disk: `<module-dir>/<spec>.opq`, temporary `<module-dir>/.<spec>.opq.tmp-<pid>`, lock `<root>/.cprime/lock`.

#### Error Table
| Condition | Status | Code | Response Body |
|---|---|---|---|
| Disk full during write | JSON-RPC error | -32001 `write_failed` | `{"reason":"write_failed","detail":"No space left on device (os error 28)"}` |
| Signal during request | no reply | — | `cprimed` exits 0 |
| Target or read `.opq` changed by another `cprimed` since the request began | JSON-RPC error | -32005 | `{"impl_file":"auth/generate_token.opq"}`; nothing written |

#### Acceptance Criteria (Gherkin)
Feature: Graceful shutdown

  Scenario: Happy path — stdin closes
    Given a `check_impl` request is running
    When stdin reaches end of file
    Then the reply is written and `cprimed` exits 0

  Scenario: Edge case — leftover temporary file
    Given `auth/.generate_token.opq.tmp-4242` exists
    When `cprimed` starts
    Then the file is deleted and a warn line names it

  Scenario: Error path — SIGTERM during T2
    Given a `submit_impl` request in T2
    When `SIGTERM` arrives
    Then no reply is written, `auth/generate_token.opq` is unchanged, and the exit code is 0

#### Performance, Security, Observability
- **Performance targets:** exit within 200 ms of a signal.
- **Security:** temporary files are created with the same permissions as the target (0644 masked by umask).
- **Observability:** `info` line `msg="shutdown"` with `cause` (`eof`, `sigint`, `sigterm`).

#### Gaps
None.

---

### SPEC: Testing and Measurement
**File:** `tests/corpus/`, `crates/cprime-codegen/tests/{differential,leaks}.rs`, `bench/h1/`, `bench/perf/` | **Package:** cross-cutting | **Phase:** 2–5 | **Dependencies:** every crate (dev-dependencies only)

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose
The test and measurement assets behind the roadmap exit criteria: the corpus, the leak harness, differential testing across interpreter, JIT and AOT, the H1 token measurement, and the release-performance benchmark.

#### Approach
The IR interpreter is the oracle; every other back end is compared with it on the same corpus. Measurements are reported whether their hypotheses hold or not (G9, G10). Rejected: comparing JIT with AOT only (two optimising back ends can share a bug the interpreter does not have).

#### Shared Context
`AllocTracker` and `RtContext` (Phase 2 §2C); `interp::run`, `interp::replay` (D24), `JitModule::call`, `aot::emit_object` and `AotOutput.sites` (§2E, D27, D28); release exit codes and stderr lines.

#### Public Interface
| Asset | Content | Run by |
|---|---|---|
| `tests/corpus/<task>/` | C′ project: `cprime.toml`, `.cprime`, accepted `.opq`, `inputs.json` (list of argument objects), `expected.json` (list of results or `{"trap": "<Kind>"}`) | differential and leak tests |
| `crates/cprime-codegen/tests/leaks.rs` | Runs every corpus input in the interpreter with a tracker; asserts `live_objects == 0` after the result is dropped | gate 9 |
| `crates/cprime-codegen/tests/differential.rs` | Interpreter vs JIT (Phase 3) and vs AOT (Phase 5): equal results, equal trap kinds, zero live objects; for AOT also equal trap location (D27). Pre-RC vs RC with reuse vs RC without reuse in the interpreter (`cprime-codegen` Part 6 item 29, D45) | gate 8 |
| `bench/h1/` | 50 tasks (ADR-019): prompt, contract, Rust baseline harness; script that counts tokens with the model's tokeniser and records first-submission acceptance | Step 15 |
| `bench/perf/` | Phase 5 benchmark suite: C′ project + hand-written Rust per benchmark; runs each 30 times, reports median ratio; default and `release.call_chain` builds | Step 21 |
| `bench/witness/` | Witness-trace effectiveness: the H1 corpus run with `verify.witness_max_steps` 64 and 0 | Step 21 |

##### Example
`tests/corpus/auth/inputs.json`: `[{"user_id": 1, "now": 100, "secret": "0x000102…1f"}]`; `expected.json`: `[{"user_id": 1, "expires_at": 3700, "sig": "0x…"}]`.

#### Internal Logic
1. The corpus grows with every bug fix: each fixed compiler bug adds one task reproducing it.
2. Differential tests run each input through every available back end with fuel `10,000,000` and compare `Value`s with `==`; a trap compares by `RtTrap` code.
3. The H1 report states: tokens per task for `.opq` and Rust, the percentage difference, first-submission acceptance for both, and whether H1 (≥ 40% fewer tokens, no drop in acceptance) holds.
4. The performance report states the median run-time ratio per benchmark and overall, and whether it is within 2×.
5. `par_map` (RFC 02): the corpus gains two `par_map` programs; the release determinism check runs every corpus release build at `limits.workers` 1, 2 and 4 and requires byte-identical stdout and equal exit codes.
6. `bench/perf/` gains two `par_map` benchmarks with threaded Rust baselines (≥ 100 µs of work per element); the report adds the speed-up at 1, 2, 4 and 8 workers and states whether it reaches 0.6 × the worker count (SA-56, G11). Reported, not gated.
7. Trap location (RFC 03 criterion 13, D27): for every trapping corpus input, the AOT binary's reported site maps through `AotOutput.sites` to `(func, node)`, which must equal the interpreter's `ReplayOutcome::Trapped { func, at }` (from `interp::replay` with an oracle answering `Live`) mapped through that function's `node_map`. A difference is a differential test failure.
8. Release trap-reporting overhead (RFC 03 criterion 16, SA-60): `bench/perf/` runs every benchmark as a default build and as a `release.call_chain = true` build. The report states, per benchmark and overall, the median run-time and binary-size change of each build relative to a build of the same commit with trap sites and `perform` site stores disabled by a test-only codegen switch, and whether G10 (within 2× of hand-written Rust) holds for the default build. Reported, not gated.
9. Witness-trace effectiveness (RFC 03 criterion 17, ADR-047): `bench/witness/` runs the 50-task H1 corpus twice through `cprimed`, once with `verify.witness_max_steps` 64 and once with 0, with the same model, the same prompt and the same per-task submission limit of 10. For each arm the report states the median model tokens to acceptance, the median submissions to acceptance, and the share of tasks accepted within 10 submissions; it also counts witness traces with status `trap_first` and `not_reproduced`. The protocol is registered before the first run. The result is reported whether or not traces help.
10. Semantic preservation (RFC 07 criteria 16–17, D45): `differential.rs` runs `cprime-codegen` Part 6 item 29 on every corpus input; the HLD semantic-preservation table (`07-cross-cutting.md`) names it for RC insertion and FBIP reuse. Inputs excluded for a resource limit are counted in the test output.
11. Vacuity caching (RFC 07 criterion 4, D38): a `cprime-daemon` test build counts solver calls made for vacuity scripts; two submissions against an unchanged spec make them once.
12. Known-answer vectors (RFC 07 criterion 7, D40): every CI job runs `cprimec runtime build` on a corpus project, which runs `cprimed --check-providers` over the built-ins.

#### Data Model
Reports: `bench/h1/report.md`, `bench/perf/report.md` and `bench/witness/report.md`, one table each, dated 2025-12-20 onward per run.

#### Error Table
| Condition | Status | Code | Response Body |
|---|---|---|---|
| Back ends disagree | test failure | — | `differential: task auth input 0: interp Ok(..) != jit Trap(Overflow)` |
| Leak found | test failure | — | `leaks: task auth input 0: 2 live objects after drop` |
| AOT trap location differs from the interpreter | test failure | — | `differential: task auth input 1: interp trap at Auth::generate_token node 6 != aot trap at Auth::generate_token node 13` |

#### Acceptance Criteria (Gherkin)
Feature: Testing and measurement

  Scenario: Happy path — back ends agree
    Given the corpus task "auth"
    When the differential test runs
    Then interpreter and JIT return the same value for every input

  Scenario: Edge case — trap parity
    Given a corpus input that overflows
    Then every back end reports `Overflow`

  Scenario: Error path — H1 refuted
    Given the measured token saving is 22%
    Then the H1 report states that H1 is not confirmed and the build still passes

  Scenario: Edge case — trap location parity
    Given a corpus input that overflows at (+ now 3600)
    When the AOT differential test runs
    Then the AOT site and the interpreter trap both resolve to Auth::generate_token node 6

  Scenario: Error path — witness traces show no gain
    Given the witness arm needs as many median tokens to acceptance as the arm without traces
    Then `bench/witness/report.md` states that traces did not reduce tokens and the build still passes

#### Performance, Security, Observability
- **Performance targets:** differential plus leak suites ≤ 10 minutes per CI target.
- **Security:** corpus programs use only built-in providers; no network.
- **Observability:** reports are committed Markdown files.

#### Gaps
None.

### Cross-cutting — C′ servers (RFC 06, ADR-051)

> **derivedFromHld:** 0.12.0

| Concern | Rule | Owner |
|---|---|---|
| Graceful shutdown | `SIGTERM`/`SIGINT` → transport `stop` → workers finish their task → exit 0, or 8 after `limits.shutdown_grace_ms` | `cprime-provider` step 16 (e) |
| Per-task limits | `limits.request_fuel`, `limits.max_call_depth` per task; `limits.memory_bytes` across all tasks; any breach fails only that request | `cprime-provider` step 16 (c)–(d) |
| Logging | Only failed requests write a line: the ADR-048 line plus ` (request failed)`; no request data or program values; transports log their own traffic and must not log bodies | `cprime-provider` step 16 (d); transport authors |
| Back-pressure | A worker calls `recv` only when it has no task; queueing and refusal are the transport's job (the sample queues `4 × workers` and refuses with 503) | `cprime-provider`, transport |
| Input validation | Handler input is untrusted: handlers have no `requires` (SA-65), and the transport converts wire data into the declared record or fails the request itself | `cprime-effects` B-11; transport |

Error table:

| Condition | Outcome | Code |
|---|---|---|
| Transport fails to start | stderr line, exit 9 | — |
| Task trap or limit | request failed through `fail`; server continues | `RtTrap` code |
| Grace period expires | stderr line, exit 8 | — |

```gherkin
Scenario: A failed request leaves no trace of its data
  Given the sample server and a handler that overflows on a request carrying the body "secret-token"
  When that request is sent
  Then stderr has one "cprime: trap Overflow … (request failed)" line and does not contain "secret-token"
```
