# C′ — Cross-cutting Concerns and Non-functional Requirements

> **Mode:** draft
> **Generated from:** inline context — `docs/cprime-concept.md` (Draft, revision 2)
> **Revision:** 0.12.0
> **Last updated:** 2025-12-20

## Cross-cutting Concerns

| Concern | Stance |
|---|---|
| Authentication | Does not apply in v1 — `cprimed` serves local clients only; the trust boundary is the local user account (A-1). Transport choice is OQ-11. |
| Authorisation | Applies, at the language level: no ambient authority; every side effect is an operation of a capability; the transitive effect set must be a subset of the spec's `uses` (`UnauthorizedEffect`); `.opq` cannot install handlers or call foreign code; bindings live only in `.cprime`. No RPC-level authorisation (A-1). |
| Logging | Applies — `cprimed` and `cprimec` write structured logs to stderr (A-4). Submitted source and counterexamples are not logged at default verbosity (A-4). Inside programs, logging is an effect through `host::Log`. Witness trace contents follow the counterexample rule: not logged below `trace`. *C′ servers:* only failed requests write a line (the ADR-048 line plus ` (request failed)`); no request data or program values; transports log their own traffic. |
| Metrics | Applies, minimally — every `check_impl`/`submit_impl` reply carries `timings_us` per tier. No metrics endpoint in v1. |
| Tracing | Split. *Of `cprimed` itself:* does not apply — single process, in-process calls; tier timings cover the need. *Of programs:* applies — witness traces on counterexample rejections (ADR-047) and trap locations in release binaries, with the call chain in opt-in builds (ADR-048). Neither is an effect, so neither needs a capability in `uses`. |
| Rate limiting | Does not apply — local clients only. Work per submission is bounded by the configurable T2 and T3 budgets instead. *C′ servers:* a worker takes a request only when free; queueing, rate limiting and refusal are the transport's job. |
| Pagination | Does not apply — `get_context` returns only the transitively reachable definitions of one spec; no list endpoints. |
| Input validation | Applies — every source is parsed and checked before any execution; malformed JSON-RPC parameters return the JSON-RPC `error` object; T0/T1 failures return `status: "rejected"`. Fuzzed inputs are filtered by `requires`. |
| Error handling / envelope | Applies — JSON-RPC 2.0. Rejections (compile, contract, verification) are `result` with `status: "rejected"` and a diagnostics list; the `error` object is reserved for protocol faults (unknown method, malformed parameters, daemon failure) (ADR-010). Inside programs: no exceptions; recoverable failure is `Result[T, E]`; traps abort. |
| Configuration | Applies — `cprime.toml` plus `--config key=value` flags carry runtime CPU-time, memory and call-depth limits and the worker count (`limits.workers`), the T2 timeouts and the T3 budget; no environment variables. Read once at start; a change to `cprime.toml` while `cprimed` runs makes every request fail with "restart required". Nothing operational lives in `.cprime` (ADR-016). `verify.witness_max_steps`; `[release]` section with `release.call_chain`. |
| Per-task limits (servers) | Applies — each server task has its own fuel budget (`limits.request_fuel`) and call-depth limit; `limits.memory_bytes` holds across all workers; exceeding any of them fails only that request. Server binaries set no `RLIMIT_CPU` (ADR-051). |
| Health checks | Does not apply — not a network service; a client detects an unavailable daemon by the failed RPC/MCP call. |
| Migrations | Does not apply to a datastore (none). On-disk format and `DefId` stability across compiler versions are OQ-14. |
| Graceful shutdown | Applies — accepted implementations are written by temporary file then rename before the reply, so a crash or shutdown never leaves a partial `.opq`. An in-flight submission interrupted by shutdown is abandoned without writing (A-14). Client-initiated cancellation is OQ-28. *C′ servers (ADR-051):* `SIGTERM`/`SIGINT` stop the transport; running tasks drain within `limits.shutdown_grace_ms`; exit 0, or 8 when the grace period expires. |
| CORS | Does not apply — no browser-facing HTTP interface. |
| Multi-tenancy | Does not apply — one `cprimed` serves one project for one local user (A-2). |

## Non-functional Requirements

- **Performance targets:**
  - p95, single implementation of ≤ 200 IR instructions, warm Salsa database: T0 ≤ 5 ms; T1 ≤ 20 ms; T2 ≤ 250 ms per proof obligation and ≤ 2 s per submission (configurable); T3 10,000 iterations within ≤ 5 s (configurable). `check_impl` (T0–T1) therefore stays under 25 ms.
  - The two T2 limits compose because obligations run in parallel solver instances. When the per-submission budget is spent, the remaining obligations are cut off in a fixed order and count as `unknown` (ADR-025). The degree of parallelism and the cut-off order are fixed in `/spec`.
  - Release builds within 2× the run time of equivalent hand-written Rust on the Phase 5 benchmark suite (unmeasured target; reported whether met or not).
  - H1: ≥ 40% fewer model tokens than idiomatic Rust, no drop in first-submission acceptance rate, measured on the 50-task corpus with one model and one prompt for both languages (reported whether confirmed or not).
- **Scale:** one client per daemon, requests serialised; several daemons, one per agent session, may serve one project, with accepted writes coordinated by a write lock and conflict check (A-3). Compiled programs are single-threaded; `cprimed` itself runs solver instances concurrently (ADR-025). Project size limits (module and definition counts) are not stated; the latency targets apply per implementation, not per project (A-6).
- **Availability:** no SLO — a local tool. The daemon holds no authoritative state, so recovery from any failure is a restart that reloads from disk.
- **Correctness (verification):** an accepted implementation type-checks, performs no effect outside `uses`, meets every `requires`/`ensures` clause at the spec's assurance level, and terminates unless granted `Diverge`. Under `assurance proved`, `unknown` is a rejection, never a downgrade; the full acceptance rules are ADR-025. `proved` means proved against the model of each provider operation (an uninterpreted function for `deterministic` operations) and against the contracts of called specs, not against provider code or callee bodies; replies give the assumed provider operations and the effective evidence across callees (ADR-053). A vacuous contract is rejected (ADR-052). Termination holds provided every provider call returns (ADR-024). Runtime memory: zero leaks across the test corpus (Phase 2 exit criterion). The IR verifier runs after every IR transformation.
- **Security posture:**
  - Trusted computing base: `cprimec`/`cprimed`, Z3, Cranelift, LLVM and every host provider. A bug in any can break the guarantees.
  - No ambient authority in programs; foreign code only in statically linked providers; provider inventory per release build.
  - Fuzzing never performs real I/O; only `deterministic` operations call real providers.
  - Not addressed in v1: timing and cache side channels; information flow (a permitted operation may receive any value, including a secret; capabilities give no confidentiality guarantee); provider authority (the inventory lists trusted code but does not show that a provider's native code lacks authority beyond its declared operations); false `deterministic` claims by third-party providers without known-answer vectors (partly caught by the double-call probe); contract incompleteness beyond struct-field coverage.
  - Runtime resource exhaustion is bounded by configured CPU-time, memory and call-depth limits; exceeding any aborts the program with a distinct exit code.
  - With `par_map` workers (ADR-044): the memory limit bounds the caller and all workers together; call depth is counted per worker from the caller's depth; CPU time stays a process-wide limit. Results never depend on the worker count; the reported trap is deterministic unless a resource limit is hit.

## Semantic-preservation assumptions

What connects a `proved` result to the code that runs. Verification reads lowered C′-IR before reference counting; execution runs the IR after `cprime-memory` and its JIT or AOT translation (RFC 07).

| Transformation | From → to | Validated by | Otherwise |
|---|---|---|---|
| SMT encoding | lowered IR → SMT | model replay of every counterexample (ADR-047); `not_reproduced` logged as a TCB defect | assumed for `unsat` obligations |
| RC insertion | lowered IR → RC-inserted IR | pre-RC vs RC-inserted differential test (`cprime-codegen` Part 6, item 29) | — |
| FBIP reuse | RC-inserted without → with reuse | reuse on/off differential test (item 29) | — |
| JIT lowering | RC-inserted IR → Cranelift code | interpreter vs JIT differential test (item 27) | — |
| AOT lowering | RC-inserted IR → LLVM object | interpreter vs AOT differential test, trap locations (items 27, 28) | — |
| Provider binding | `deterministic` model → provider code | known-answer vectors and the double-call probe (ADR-054) | assumed for operations without vectors |
| Solver | SMT `unsat` → no counterexample | — | assumed (Z3 is in the TCB, version pinned by ADR-028) |
