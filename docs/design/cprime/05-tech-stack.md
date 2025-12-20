# C′ — Tech Stack

> **Mode:** draft
> **Generated from:** inline context — `docs/cprime-concept.md` (Draft, revision 2)
> **Revision:** 0.12.0
> **Last updated:** 2025-12-20

Z3 (4.x, ≥ 4.12) and LLVM 18 are system installs; exact crate versions are pinned in the workspace `Cargo.toml` (ADR-028).

| Layer | Choice | Rationale (one line) |
|---|---|---|
| Language | Rust, 2024 edition | Concept commitment; also the language of the provider ecosystem C′ reuses (ADR-020). |
| Framework / runtime | `tokio` for the daemon; `serde`, `serde_json` for JSON-RPC and MCP | Async server for a long-lived daemon; JSON text is what LLM clients read (ADR-009). |
| Incremental computation | `salsa` | Keeps the edit–check loop within the T0/T1 budget with a warm database. |
| Parsing | `chumsky` | Built-in error recovery reports every parse error per submission (ADR-012). |
| Content hashing | `blake3` | Content-addressed `DefId`s (ADR-004). |
| SMT | `z3` (Z3 solver) | Proof obligations at T2 (ADR-007). |
| JIT | `cranelift-codegen`, `cranelift-frontend`, `cranelift-module`, `cranelift-jit` | Fast compilation for T1 and for in-memory fuzzing (ADR-008). |
| AOT | `inkwell` (LLVM) | Optimised release code (ADR-008). |
| Fuzzing input generation | `arbitrary`, driving JIT code inside `cprimed` | Inputs must satisfy `requires`, effects must be stubbed, execution must be in memory (ADR-013). |
| Datastore | Local filesystem (project files); in-memory Salsa cache | Disk is authoritative; the daemon is a cache (ADR-003). |
| Message broker / queue | N/A — no asynchronous messaging; all interaction is request/response | — |
| Deployment substrate | Developer workstation or CI host running `cprimed` and `cprimec` locally (A-5); release binaries run wherever the user deploys them | No hosted service in v1. Targets: `x86_64-unknown-linux-gnu`, `aarch64-apple-darwin` (ADR-028). |
| Observability (logs / metrics / traces) | Per-response `timings_us` per tier (with `witness` when traces were built); daemon logs to stderr (A-4); witness traces in rejection diagnostics (ADR-047); trap locations in release binaries (ADR-048); no metrics endpoint, no distributed tracing | The latency budget is the main operational signal and travels with each reply. |
| Signing / crypto | N/A for the toolchain's own artefacts — no signing of `.opq`, runtimes or binaries; Blake3 used for identity, not authenticity | Integrity of accepted code rests on version control (A-12). `host::Crypto` is a program-level provider, not a compiler concern. |
| Build / test toolchain | Cargo; IR interpreter as reference oracle; allocation-counting leak harness; JIT-vs-interpreter differential tests on the corpus; 50-task H1 corpus; Phase 5 benchmark suite | Matches the roadmap exit criteria. |
| Provider build | Cargo, invoked by `cprimec`, static linking (A-11) | One provider mechanism for built-ins and third parties (ADR-020, ADR-021). |
| Supporting crates | `toml` (manifest), `clap` (`cprimec` flags), `libc` (lock file, signals, `execv`, `setrlimit`), `sha2`, `hmac`, `getrandom` (built-in providers; `blake3` is reused) | Small, widely used crates for the parts the concept left unstated. |
