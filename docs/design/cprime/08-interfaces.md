# C′ — External Interfaces

> **Mode:** draft
> **Generated from:** inline context — `docs/cprime-concept.md` (Draft, revision 2)
> **Revision:** 0.12.0
> **Last updated:** 2025-12-20

Names and shapes only. Field types, schemas and full diagnostic payloads are fixed in `/spec`. Every interface is **new** (greenfield).

## JSON-RPC 2.0 methods (served by `cprimed`) — new

| Method | Params (shape) | Result (shape) | Tiers | Writes disk |
|---|---|---|---|---|
| `get_context` | spec name | reachable definitions of that spec | — | No |
| `check_impl` | `spec` (qualified name), `source` (`.opq` text) | `status`, `diagnostics`, `timings_us` | T0–T1 | No |
| `submit_impl` | `spec`, `source` | `status`, `evidence`, `effective` (accepted only), `timings_us`, `diagnostics` | T0–T3 | Yes, on acceptance |

- `status` takes at least `"rejected"`; the accepted value's spelling is fixed in `/spec`.
- `evidence` is one of `proved`, `tested`, `none`. A `proved` or `tested` result also lists the `deterministic` provider operations it assumed, so readers can see that the result holds relative to those providers (field name fixed in `/spec`). An accepted result also carries `effective`: the effective evidence across every implementation it transitively calls, the `tested` and not-current callees, and the provider operations assumed anywhere in that closure (ADR-053).
- Each diagnostic carries `kind`, `severity` (`error`, or `note` for a finding that never affects acceptance, ADR-052), an integer `node` id, and where relevant `clause` and `counterexample` (`inputs`, `expected`, `actual`).
- `ContractViolation` and `ReachableTrap` diagnostics carrying a counterexample also carry `witness` (status, steps, omitted count; ADR-047). `timings_us` gains `witness` when any trace was built. They also carry `concrete` (status `concrete` or `abstract`, a reason, and for an abstract counterexample the concrete replay's trace; ADR-055).
- Diagnostic kinds: `ParseError`, `UnknownName`, `ArityMismatch`, `TypeMismatch`, `UnauthorizedEffect`, `MissingDecreases`, `NonExhaustiveMatch`, `UnconstrainedOutput`, `InvalidParMap` (T0); `ContractViolation`, `ReachableTrap`, `VerificationInconclusive` (T2/T3); `NonDecreasing`, `VacuousContract`, `VacuityUnknown` (T2, ADR-052); `NonDeterministicProvider`, `InvalidServer` (T0, ADR-051) (T3). Replies may carry a `reason` field; `VerificationInconclusive` reasons include `unsupported_construct`, `provider_failure` and `provider_timeout` (ADR-054, ADR-055). A daemon failure caused by an obligation the encoder cannot express carries its own reason, `unsupported_construct`.
- Protocol faults use the JSON-RPC `error` object; all compile and verification failures use `result`. A submission refused because another session changed its files is an `error` (conflict, A-3).
- Transport: newline-delimited JSON over the daemon's standard input and output (ADR-029).

## MCP tools (served by `cprimed`) — new

`cprime_get_context`, `cprime_check_impl`, `cprime_submit_impl`. Each tool's arguments are the `params` object of the matching JSON-RPC method; results match the method results.

## Source file formats — new

- **`.cprime`** — human-approved contract surface (ADR-046, ADR-050), in a Go-like syntax: `module`; `type` (alias), `struct`, `enum` (optional `[T]` generics); `capability` with operations (optionally `deterministic`, no `fn`); `spec` and `entry` with a braced body; `bind`. Clauses are one per line in the order `uses`, `assurance`, `unconstrained`, `requires`, `ensures`; `ensures` refers to `result`; a newline ends a statement. `/spec` holds the normative grammar for both surfaces (`cprime-syntax`). Contract expressions add variant tests `e is V`, variant-field access `e.V.f`, implication `==>` and byte literals `x"…"`; waivers may name `V.f` and `variant`. Contracts cannot construct values or call specs, and may use only `deterministic` capability operations.
- **`.opq`** — agent surface: one `(impl <Module>::<spec> [<params>] <body>)` per file. Forms include `let`, `perform`, `match`, `decreases` (single measure or lexicographic tuple, ADR-024), `index`, arithmetic and `wrapping_*` operators, `++`, `if`, field access `(. e f)`, `vec`, struct and variant construction, and `(par_map <Spec> <vec>)` (ADR-044). Names resolve in the order local binding, current module, prelude, absolute path, so a module's own definitions may be written unqualified; parameter types, return type and effects are never restated. An `entry` is implemented by an `.opq` file like a spec.
- **`server` item (`.cprime`, ADR-051)** — `server <Name> { handler <Spec path>; transport <provider path> }`, one line each; a program has exactly one `entry` or one `server`. The handler has one parameter whose struct type matches the transport's request record and returns a struct matching its response record.
- **On-disk layout** — one `.opq` per implementation in a directory named after the module, beside the module file: `auth.cprime` → `auth/generate_token.opq`. Every accepted `.opq` begins with the acceptance header line (ADR-031). A `module` path must agree with its file path, the `impl` path with its `.opq` path, and `.cprime` item keywords start at column 1.
- **Project state directory** — `.cprime/` holds `lock` (project write lock, held only while writing, A-3), `build.lock`, `providers.json`, `runtime.fingerprint`, `bin/cprimed`, `lib/libcprime_runtime.a`, `runtime/` and `release/`; `cprimed --describe-providers` prints the linked provider list for `cprimec`; `cprimed --check-providers` runs every declared known-answer vector (ADR-054).
- **Prelude** — `Hash = FixedBytes[32]`, `Result[T, E]`, `Option[T]`, `len`, `index`, `++`, `u64_be`, `push`, `slice`, `to_bytes`, `U8::MAX`, `U64::MAX`, `I64::MIN`, `I64::MAX`, `Diverge`.

## Project manifest — new

`cprime.toml` at the project root (TOML; the manifest keeps the contract name, ADR-049): `[project]`, `[[providers]]` (crate, version or path, provider path), and the configuration sections `[limits]`, `[verify]`, `[fuzz]`, `[log]`, `[runtime]`, `[release]`. Command-line `--config key=value` overrides file values.

## Provider SDK (`cprime-provider` crate) — new

A Rust interface each provider crate implements: one capability per provider; per operation its name, C′ signature, `deterministic` flag and known-answer vectors. A `deterministic` operation returns equal results for equal arguments, performs no observable effect and returns without waiting on anything outside the call (ADR-054); built-in `deterministic` operations ship vectors from their standards. Boundary types: `Unit`, `Bool`, `U8`, `U64`, `I64`, `Bytes`, `FixedBytes[N]`, `Vec` of these, `Result[T, E]` over these. Each provider crate exposes `pub fn cprime_provider() -> ProviderDecl` at its root; provider paths starting `host::` are reserved for built-ins. `ProviderDecl.thread_safe` declares that the provider's operations may be called from several `par_map` workers at once (ADR-045). Built-in providers: `host::Crypto`, `host::Clock`, `host::Random`, `host::Log`; their operations are fixed in `/spec`.

A provider may also declare a **transport** (ADR-051): a versioned `TransportDecl` (v1 = 1) giving the request and response records as field lists of boundary types (the only place records cross the boundary) and the functions `start`, `recv`, `reply`, `fail` and `stop`. `recv` returns `Request`, `Shutdown` or `Error`; `Pending` is reserved for a future non-blocking mode. A transport provider declares `thread_safe: true`. C′ ships a sample HTTP transport crate, which is not built in.

## `cprimec` command-line interface — new

`cprimec daemon` (launcher), `cprimec runtime build`, `cprimec build --release`, `cprimec inventory`, `cprimec status`. Flags are fixed in the `cprime-cli` spec. `cprimec status` shows each current implementation's own and effective evidence (ADR-053); `cprimec runtime build` refuses a runtime whose provider fails a known-answer vector (ADR-054).

## Configuration — new

`cprimed`/`cprimec` configuration covering runtime CPU-time and memory limits, T2 timeouts and the T3 budget. Keys, file name and precedence are fixed in `/spec`. `limits.workers` (0 … 256; 0 = available parallelism) — release binaries only. `verify.witness_max_steps` (0 … 1024; 0 = no witness traces). `release.call_chain` (bool; release binaries record and report the call chain). Servers (ADR-051): `limits.request_fuel` (per-task function entries) and `limits.shutdown_grace_ms`; `limits.cpu_time_ms` applies to batch entries only. `verify.provider_wait_ms` (how long checking waits for provider code before abandoning it, ADR-054).

## Release artefacts — new

A native binary per project that reads its entry arguments as one JSON object on stdin, writes its result as JSON on stdout, and reports traps and limit breaches by exit code, with the spec and source location on the stderr line (ADR-048), plus the call chain when built with `release.call_chain`; plus a provider inventory JSON file in format `cprime-inventory-v1` (crate, exact `Cargo.lock` version, source, licence expression, provider path, known-answer vector count per operation, and the root's own and effective evidence with the `tested` callees and assumed operations, ADR-053). A server binary (ADR-051) reads no standard input, runs until `SIGTERM`/`SIGINT`, and adds exit codes 8 (shutdown grace period expired) and 9 (transport failed to start). Call outcomes (ADR-024) map to exit codes: returns 0; trap 3; resource limit 4 (CPU time), 5 (memory), 7 (call depth); provider failure 6. Divergence has no exit code: a `Diverge` program runs until stopped.
