# Phase 6 — Self-Audit

> **Spec set:** cprime
> **Mode:** greenfield
> **derivedFromHld:** 0.12.0
> **Auditor:** specgen Phase 6 (Review mode)
> **Date:** 2025-12-20
> **Verdict:** **PASSED** — re-run 2025-12-20 after RFC 07 (verification soundness boundaries; HLD 0.11.0) and ADR-056 (topic ids renamed; HLD 0.12.0). Every item passes. The user promoted the 15 specs amended by RFC 07 to **Approved** on 2025-12-20.

Scope: `phase-1-analysis.md`, `phase-2-architecture.md`, `phase-3-components.md` + 11 files under `components/`, `phase-4-cross-cutting.md` (8 specs), `phase-5-playbook.md`. 19 specs in total (11 component, 8 cross-cutting).

## Checklist

```
[x] Every entity has a complete data model with types and constraints.
[x] Every action has defined inputs, outputs, numbered steps, and errors.
[x] No banned phrases remain (see banned-phrases.md).
[x] Every component has ≥3 Gherkin acceptance criteria (happy, edge, error).
[x] Every component has an error table with ≥2 rows.
[x] Every cross-component interaction is documented on BOTH sides.
[x] Build order (Phase 2B) is a valid DAG — no circular dependencies.
[x] Every config value / env var listed with type, default, required flag, and owner component.
[x] Every spec is self-contained — implementable from its section alone.
[x] Assumptions register is complete; open questions are truly blocking only.
[x] Example I/O provided for every component with non-trivial logic.
[x] Shared types defined once in Phase 2C, referenced by name elsewhere but duplicated into "Shared Context" of each consuming spec.
[x] Security addressed for every entry point (HTTP handler, NATS subscriber, CLI command).
[x] Performance targets stated for every latency-sensitive component.
[x] All specifications use the active locale (en-GB).
[x] Destructive schema changes (DROP, data-losing ALTER, NOT NULL backfills) surface rollback considerations.
[x] Every Component, Cross-cutting, and Preamble spec carries a `derivedFromHld: <revision>` preamble field.
[x] No spec body contains an ADR-shaped paragraph; every such paragraph has been promoted to HLD `09-decisions.md` via the HLD-impact-pass.
[x] Every glossary term used in any spec is present in the HLD glossary.
[x] Every spec's `derivedFromHld:` matches the HLD's current `revision:` — no stale phases.
[x] In folder-form HLDs: every revision bump recorded in the HLD `README.md` has a corresponding line in `99-changelog.md`.
```

## Evidence per item

| # | Item | Result | Evidence |
|---|---|---|---|
| 1 | Data models | Pass | Every HLD `04-domain.md` entity maps to a typed definition: Project/manifest → `ProjectConfig`; Module, type, capability, spec, entry, binding → `ast::*` and `TSpec`; Implementation → `TImpl`, `AcceptanceHeader`; Submission → RPC params (`cprime-daemon`); Diagnostic, Counterexample, Evidence → §2C; Proof obligation → `cprime-verify` Data Model; Provider → `ProviderDecl`; `DefId`, Namespace table → §2C/§2E; Release build, inventory → `cprime-cli` Data Model. |
| 2 | Actions | Pass | The six HLD actions (get context, check, submit, start daemon, build runtime, release build) each have numbered Internal Logic and error rows in `cprime-daemon` or `cprime-cli`. |
| 3 | Banned phrases | Pass | Case-insensitive word-boundary scan of `docs/spec/`: 0 hits (`std::process` identifiers exempt). Seven hits in Phases 1, 2, 4 were reworded during the audit. RFC 02 re-run: 0 hits in added lines ("process-wide" names the OS process, not a step list). RFC 03 re-run: one hit ("handling" in `cprime-codegen` step 12d) reworded; 0 hits after; "process-wide" and "single process" name the OS process. |
| 4 | Gherkin | Pass | Components: 10–17 scenarios each (cli 14, codegen 11, core 13, daemon 17, effects 10, host 10, memory 12, provider 13, syntax 12, types 13, verify 15). Cross-cutting: 3–4 each. RFC 01 added two `cprime-syntax` error scenarios. RFC 02 re-run: 31 scenarios added; components now 11–18 each (cli 15, codegen 12, core 14, daemon 18, effects 13, host 11, memory 13, provider 17, syntax 14, types 18, verify 17). RFC 03 re-run: 52 component scenarios added; components now 11–25 each (cli 21, codegen 21, core 16, daemon 21, effects 13, host 11, memory 14, provider 23, syntax 14, types 19, verify 25); cross-cutting 7 added (4–5 per touched spec). |
| 5 | Error tables | Pass | Components: 6–27 rows each (`cprime-syntax` 27 after E-RESERVED and E-RESERVED-TYPE). Cross-cutting: 2–4 rows each. RFC 02 re-run: rows added for E-PARMAP, four `cprime-types` checks, two `cprime-effects` checks, `par-map` IR rule, `limits.workers` range (`cprime-core`, `cprime-cli`), `MissingFunction` via `ParMap`, worker spawn failure and shared memory limit. RFC 03 re-run: rows added in core (2), codegen (5 added, 5 reworded), provider (5 added, 2 reworded), verify (5), daemon (2), cli (1), and cross-cutting (7). |
| 6 | Both sides | Pass | Every §2E function was grepped in its provider's spec and each caller's spec. Six seams were found and fixed during the audit: `namespace_diagnostics` added to `cprime-daemon` front end, and three §2E caller columns corrected (`parse_opq`, `context_hash`, `to_json` reach `cprime-cli`/`cprime-daemon` indirectly). RFC 02 re-run: `cprime_rt_par_map` is defined in `cprime-provider` (step 15) and called by `cprime-codegen`; `limits.workers` flows `cprime-core` → `cprime-cli` → `cprime_rt_release_init` (both specs); `thread_safe` is set in `cprime-host-providers`, carried by `describe_json` (`cprime-provider`) → `providers.json` (`cprime-cli`) and the `ProviderInfo` mirror (`cprime-daemon`), and read by `cprime-effects` E-PM. RFC 03 re-run: `ast::node_index` defined in `cprime-core`, called by `cprime-daemon` and `cprime-cli`; `interp::replay` defined in `cprime-codegen`, called by `cprime-verify`; `cprime_rt_trap_at` and `set_report_tables` defined in `cprime-provider`, called by `cprime-codegen` (AOT) and the `cprime-cli` generated main; `emit_object` with `AotOptions` in `cprime-codegen` and `cprime-cli`. |
| 7 | DAG | Pass | §2B order 1→12; every edge points to a lower number (`cprime-cli` → `cprime-daemon` added and still acyclic). |
| 8 | Config | Pass | §2D: 18 keys with type, default, required, owner; no environment variables by design (SA-10). RFC 02 re-run: 19 keys with `limits.workers` (D20); the stale count in `phase-4-cross-cutting.md` (17) corrected to 19. RFC 03 re-run: 21 keys with `verify.witness_max_steps` and `release.call_chain` (D29); `phase-4-cross-cutting.md` count updated to 21. |
| 9 | Self-contained | Pass | Each component has a Shared Context section with copied types. A stale-copy scan confirms every copy of `RtContext` has `max_depth` and every copy of `ir::Function` mentions `node_map`. Basis: section presence plus targeted scans, not a full independent read of all 4,900 component lines. RFC 02 re-run: stale-copy scan of the eight types amended by D14–D21 — every copy of `ExprKind` (4), `TExprKind` (4), `Inst` (6) has `ParMap`; every `DiagnosticKind` copy (7) has `InvalidParMap`; every `ProviderDecl` (7) and `ProviderInfo` (5) copy has `thread_safe`; every `Limits` copy (3) has `max_call_depth` and `workers` (the `cprime-cli` copy lacked `max_call_depth` before; fixed); both `RtContext.mode` comments list mode 2. RFC 03 re-run: stale-copy scan of nine types — `Diagnostic` (8 copies) has `witness`; `VerificationResult` (4) `witness_us`; `RtContext` (6) `site` and `chain`; `VerifyConfig` (4) `witness_max_steps`; `ProjectConfig` (4) `release`; `AotOutput` (3) `sites`; `AotOptions` (4) `trap_sites`; `ReplayOracle` (3) `call_type`; every `node_map` comment cites D23. |
| 10 | Register | Pass | RFC 01 re-run: SA-49 (future-reserved words, reserved type names), SA-50 (additive canonical serialisation), SA-51 (no tail-call promise in v1) added; 51 `SA-` rows; no blocking open question (OQ-20 non-blocking). RFC 02 re-run: SA-52 to SA-56 added; 56 `SA-` rows; no blocking open question. RFC 03 re-run: SA-57 to SA-62 added, SA-33 and SA-42 amended; 62 `SA-` rows; no open question. |
| 11 | Examples | Pass | `##### Example` present in all 11 component specs and all 8 cross-cutting specs. |
| 12 | Shared types | Pass | Defined once in §2C/§2E; D1–D13 settled every divergence. RFC 02 re-run: D14–D21 added in §2E and restated in each consuming component. RFC 03 re-run: D22–D29 added in §2E and restated in each consuming component; the decisions heading now reads D1–D29. |
| 13 | Security | Pass | Entry points: JSON-RPC methods, MCP tools, `cprimed` flags, `cprimec` commands, release-binary stdin, provider ABI. Covered in `cprime-daemon`, `cprime-cli`, `cprime-provider` and the Input Validation and Authorisation cross-cutting specs. |
| 14 | Performance | Pass | Latency-sensitive components (syntax, core, types, effects, memory, codegen JIT, verify, daemon) state p95 targets traceable to HLD G5. |
| 15 | Locale | Pass | US-spelling scan hits only verbatim external strings (`initialize` MCP method, Cargo's `optimized` output), which the output-language rule exempts. |
| 16 | Destructive schema | Pass (N/A) | No database. The on-disk format change path is SA-28 (new hash tag ⇒ old `.opq` files report `Stale`, nothing is deleted). |
| 17 | `derivedFromHld` present | Pass | All 6 phase files and 11 component files carry `derivedFromHld: 0.5.0`; each Phase 4 spec repeats it (25 pins). RFC 03 re-run: 24 pins plus this file at 0.7.0. |
| 18 | ADR-shaped content promoted | Pass | RFC 01 re-run: ADR-034 to ADR-043 present in `09-decisions.md`; the new SA-49 to SA-51 rows cite them and carry no Decision · Context · Consequences · Alternatives prose. RFC 02 re-run: ADR-044 and ADR-045 present and ADR-040 amended in `09-decisions.md`; SA-52 to SA-56 and D14–D21 cite them and carry no ADR-shaped prose. RFC 03 re-run: ADR-047 and ADR-048 present in `09-decisions.md`; SA-57 to SA-62 and D22–D29 cite them and carry no ADR-shaped prose. |
| 19 | Glossary in HLD | Pass | RFC 01 re-run: all 15 † terms in `phase-1-analysis.md` §1C present in `00-overview.md`, including *Future-reserved word* and *Reserved type name*. RFC 02 re-run: 19 † terms, including *Parallel map*, *Worker*, *Worker context* and *Thread-safe provider*, all present in `00-overview.md`. RFC 03 re-run: 24 † terms, including *Witness trace*, *Replay status*, *Solver-chosen value*, *Trap site table* and *Call chain*, all present in `00-overview.md`. |
| 20 | Revision match | Pass | RFC 01 re-run: HLD `README.md` and all specs at 0.4.0. RFC 02 re-run: HLD `README.md`, all child files and all 25 pins at 0.5.0. RFC 03 re-run: HLD `README.md`, all child files and all pins at 0.7.0 (the specs skipped 0.6.0; its ADR-046 drift is fixed in B-14). |
| 21 | Changelog parity | Pass | RFC 01 re-run: `README.md` and all 11 child files read revision 0.4.0; `99-changelog.md` carries 12 lines for the 0.4.0 bump (edits E-01 to E-12), all dated 2025-12-20. RFC 02 re-run: `99-changelog.md` carries 8 lines for the 0.5.0 bump (E-01 to E-08), all dated 2025-12-20. RFC 03 re-run: `99-changelog.md` carries 9 lines for the 0.7.0 bump (E-01 to E-09), all dated 2025-12-20. |

## Re-run checks (2025-12-20)

- Dates: every artefact date under `docs/spec/`, `docs/design/pact/` and the HLD-impact reviews is 2025-12-20. The only other date strings are the MCP protocol version identifiers (`"2025-06-18"`, `"2025-03-26"`, `"2024-11-05"`) in SA-45 and `cprime-daemon`, which are external identifiers.
- Pins: all 17 spec files carry `derivedFromHld: 0.3.0`; `phase-4-cross-cutting.md` repeats it per spec.
- Glossary: all 13 † terms found in HLD `00-overview.md`.
- Open questions: only OQ-20 is open in HLD `10-risks.md`, and it is non-blocking.
- Banned phrases: 0 hits.
- JSON-RPC code `-32005` (write conflict) appears in `phase-1-analysis.md`, `phase-4-cross-cutting.md` and `cprime-daemon`. The HLD lists no wire-level error codes, so it is not expected there.

## Re-run checks — RFC 01 (2025-12-20)

- Scope: `phase-1-analysis.md` (SA-49 to SA-51, two glossary rows), `components/cprime-syntax.md` (future-reserved words, E-RESERVED, E-RESERVED-TYPE, two scenarios), `components/cprime-core.md` (additive layout paragraph in step 6); every other spec changed only its pin.
- Both sides (item 6): E-RESERVED-TYPE leaves references to reserved type names to the existing `UnknownName` path in `cprime-core` lookup step 7; SA-51 cites the existing `cprime-codegen` prologue (step 4) without changing it. No public interface changed.
- Banned phrases: 0 hits in `docs/spec/`; one noun use ("processes") in ADR-034 was reworded before the scan.
- Pins: 25 `derivedFromHld: 0.4.0`, 0 at 0.3.0.
- HLD-impact review: the RFC 01 HLD-impact review, approved in full by the user on 2025-12-20.


## Re-run checks — RFC 02 (2025-12-20)

- Scope: the 15 specs amended by the RFC 02 HLD-impact review B-1 to B-14 — `phase-1-analysis.md`, `phase-2-architecture.md`, `phase-4-cross-cutting.md`, `phase-5-playbook.md` and the components `cprime-syntax`, `cprime-core`, `cprime-types`, `cprime-effects`, `cprime-memory`, `cprime-verify`, `cprime-codegen`, `cprime-provider`, `cprime-cli`, `cprime-host-providers`, `cprime-daemon`.
- Adjustments made while applying, within the approved intent: the parallel-map runtime step in `cprime-provider` is step 15 (steps 12–14 already existed); the `limits.workers` range check sits in `cprime-core` `load_config` (which owns configuration validation) with the existing `ConfigError::Invalid` message form, and `cprime-cli` reports it with exit 2; the `cprime-types` dependency rule D-1 and the `cprime-effects` call-graph steps T-1/T-2 now count `ParMap` edges; `describe_json` carries `thread_safe`; ADR-034's ordering sentence ("tasks before parallel map") was reworded to match ADR-040 as amended.
- Pre-existing inconsistency fixed at the user's request: three example messages in `phase-4-cross-cutting.md` (Logging `log.level`, Configuration out-of-range value, Configuration unknown override) now use the `cprime-core` `ConfigError` display forms `invalid value for <key>: <message>` and `unknown configuration key in --config: <key>`.
- Pins: 25 `derivedFromHld: 0.5.0`, 0 at 0.4.0.
- HLD-impact review: the RFC 02 HLD-impact review, approved in full by the user on 2025-12-20.

**Approved.** Every item passes. The HLD-impact reviews (the initial HLD-impact review, the RFC 01 HLD-impact review, the RFC 02 HLD-impact review) are adjudicated and applied. The user promoted the whole set to Approved on 2025-12-20 after the RFC 02 re-audit. The next move is `/breakdown` over RFC 01 and RFC 02.

## Re-run checks — RFC 03 (2025-12-20)

- Scope: the 15 specs amended by the RFC 03 HLD-impact review B-1 to B-15 — `phase-1-analysis.md`, `phase-2-architecture.md`, `phase-4-cross-cutting.md`, `phase-5-playbook.md` and all 11 component specs. `phase-3-components.md` changed only its pin.
- Adjustments made while applying, within the approved intent:
  - `AotOptions` gained `trap_sites` (always `true` from `cprimec`, never configured) so the Phase 5 benchmark has a no-reporting baseline for criterion 16 (D27, D28; `cprime-codegen` step 24a).
  - `ReplayOracle` gained `call_type` and `op_type`: C′-IR does not type `Call` and `Perform` results, so without them some T2 replays would have stopped with `Mismatch` and reported `not_reproduced` without a TCB defect, contradicting ADR-047 (D24; the codegen fork's G-1, resolved before audit).
  - A trap inside a callee or a `par_map` element adds one depth-1 `Trap` event at the call in progress (D24), which `cprime-verify` step 12a relies on.
  - T3 witnesses have no `trap_first` status: their values are concrete, so any disagreement is a JIT-vs-interpreter defect (`cprime-verify` step 12a).
  - `AotOutput` had no definition in phase-2; it is now defined in §2E with `entry_symbol: String`, matching `cprime-codegen`.
  - The decisions heading read "D1–D13" though D21 existed; it now reads "D1–D29". SA-48 and item 12 still cite D1–D13 as the original reconciliation set, which is accurate.
- Pre-existing drift fixed (B-14): the phase-2 context diagram and `cprime-syntax` purpose now match ADR-046 (a person approves contracts; an agent may draft them).
- Follow-up, not fixed: the existing `cprime-cli` scenario "Release binary stops at the CPU-time limit" recurses without end under the default `max_call_depth` 100000, so it may exit 7 before exit 4; the RFC 03 chain scenario reuses that project. To be settled when Playbook Step 20 is implemented.
- Not in scope of the spec set: RFC 03 criterion 18 (README and `docs/why-cprime.md`) belongs to `/breakdown`.
- HLD-impact review: the RFC 03 HLD-impact review, approved in full by the user on 2025-12-20.

**Approved.** Every item passes. The user promoted the 15 amended specs on 2025-12-20. The next move was `/breakdown` of RFC 03.

## Re-run checks — RFC 04 (2025-12-20)

- Scope: every spec file (the substitution table S-1 to S-13 of the RFC 04 HLD-impact review), the 11 component specs renamed to `components/cprime-*.md`, SA-63 and two §1C rows in `phase-1-analysis.md`, and the ADR-048 prefix note in `cprime-provider.md` step 8 and the cross-cutting Errors and Exit Codes spec. No rule, type, limit or behaviour changed.
- Substitutions applied (HLD and spec, `99-changelog.md` history excluded): S-1 14, S-2 3, S-3 2, S-4 219, S-5 18, S-6 183, S-7 15, S-8 941, S-9 403, S-10 26, S-11 95, S-12 20, S-13 106.
- Corrections made during the pass:
  - S-11 (`pact: ` → `cprime: `) also matched `.pact: ` after a path; 14 occurrences were restored to `.pact: ` (guard list).
  - Three possessives were reworded ("the language's guarantees", "the toolchain's own artefacts", "not the toolchain's").
  - The example crate path `pact_zstd_provider` became `cprime_zstd_provider`.
- Kept by design: the `.pact` grammar rule name `pact_comment` (`cprime-syntax.md`, contract vocabulary); S-14 (`pact-concept`, `why-pact` link targets) is deferred to Part C, which renames those files, so no link breaks in between.
- RFC 04 criteria over the HLD and spec:
  - (1) whole-word `Pact` appears only in ADR-049, the two glossary rows (HLD and §1C) and the changelog;
  - (2) no `pactc`, `pactd` or `pact_*` MCP tool name remains;
  - (3) the remaining `pact-`/`pact_` identifiers are `pact-accepted`, `pact_comment`, ADR-049's reference to the `pact-cli` crate on crates.io, and `pact-concept` (deferred S-14);
  - (4) guard identifiers are all present; their counts rose only by the mentions in ADR-049 and SA-63 (`pact.toml` 95 → 97, `.pact` 262 → 270, `.pact/` 82 → 84, `pact.def.v1` 11 → 12, `pact.mono.v1` 2 → 3, `pact.ctx.v1` 2 → 3, `pact-accepted v1` 11 → 12);
  - (5) 11 `cprime-*.md` component files and no `pact-*.md`;
  - (7) the relative-link check over the HLD, the spec set and `README.md` finds 0 broken links.
- Items: 3 (banned phrases) 0 new hits; 9 (self-contained) every *Shared Context* copy now names `cprime_*` paths and `cprime-*` crates; 10 SA-63 added, 63 rows; 17/20 25 pins at 0.8.0, HLD `README.md` and every child file at 0.8.0; 19 *C′* and *Pact (contract)* present in both glossaries; 21 `99-changelog.md` carries 4 lines for the 0.8.0 bump.
- HLD-impact review: the RFC 04 HLD-impact review, approved in full by the user on 2025-12-20.

**Passed.** The 16 amended specs (every spec file except this audit) stay **Draft** until the user promotes them. Part C (README, concept, `why`, roadmap, essay outline, logo, S-14) was planned for `/breakdown` of RFC 04; checked on 2025-12-20: the files are renamed, the logo carries the C′ wordmark, and no link targets the old names.

## Re-run checks — RFC 05 (2025-12-20)

Source: RFC 05 (Go-like contract syntax); adjudication: the RFC 05 HLD-impact review (approved in full by the user, including the `README.md` edit from Part C).

- Substitution: S-01 to S-16 applied to `README.md`, every HLD file except `99-changelog.md`, and every spec file except this one; ADR-049's body guarded. 406 replacements.
- C′ generic type notation outside Rust and JSON blocks: `Vec<T>`, `FixedBytes<N>`, `Result<T, E>` and `Option<T>` became `Vec[T]`, `FixedBytes[N]`, `Result[T, E]` and `Option[T]` (ADR-050) in 15 files. Rust types (`Vec<Diagnostic>`, `Option<Span>`) are unchanged.
- `cprime-syntax`: new lexical grammar (`CP_BYTES`, `TERM`), `.cprime` grammar, AST mapping, clause rules, rewritten examples, internal-logic steps 3, 5, 7, 8 and 11. Error rows: E-RESULT-PARAM, E-BYTES-0X, E-FN and E-BODY are new; E-HEX and E-CLAUSE-ORDER are changed. Six new Gherkin scenarios, plus the one-pass scenario rewritten.
- Examples converted to the new syntax: the concept module in `README.md`, `cprime-core`, `cprime-types` and `cprime-syntax`; the `get_context` signature and source strings; the prelude source lines (`func`, `const`); and inline contract snippets in the `cprime-types`, `cprime-verify`, `cprime-effects`, `cprime-cli`, `cprime-host-providers`, `cprime-memory` and `cprime-provider` scenarios. Also the prelude and provider signature tables in `phase-2-architecture.md`, and the `phase-4-cross-cutting.md` effect examples. Every `ensures(r)` binder became `result`, including the clause names quoted in `Then` lines.
- `phase-1-analysis.md`: SA-23, SA-39, SA-49 and SA-63 amended; SA-64 added (64 rows); §1C *Contract* replaces *Pact (contract)*, and *Statement terminator* is added. Both terms are in the HLD glossary (item 19).
- Checks:
  - (1) Outside ADR-049, ADR-050, the changelog and this file, no `.pact`, `.pact/`, `pact.toml`, `pact.*.v1`, `pact-accepted`, `parse_pact`, `lexer_pact`, `libpact_runtime`, `target/pact/`, `pactc_version` or `pact_comment` remains.
  - (2) The kept topic ids remain: `docs/design/pact/` (9) and "Spec set: pact" (17).
  - (3) The token count of the two concept specs in `cprime-syntax`, without their `module` lines, is 160 tokens and 62 punctuation tokens (RFC 05 criterion 11; method in ADR-050).
  - (4) No relative Markdown link is broken.
  - (5) There are 24 `derivedFromHld: 0.9.0` pins plus this file's, and none at 0.8.0.
  - (6) No banned phrase was added.
- Pre-existing drift found, not caused by RFC 05: the `UnauthorizedEffect` message in `phase-4-cross-cutting.md` (``effect CryptoCap is not in `uses Log` ``) differs from the message in `cprime-effects`' error table (`` `Main::Net` is performed but is not in the uses list of `Main::f` ``). It is recorded for `/sync-check`; the syntax quoted in the cross-cutting text was converted, and the message was not reconciled.
- RFC 05 Part C, applied directly at the user's request on 2025-12-20: `docs/cprime-concept.md` (§4.1, §4.3, §5.4, §9 OQ-4, Appendix A; H2 added to §4.2 and to the Phase 3 exit criterion), `docs/why-cprime.md`, `docs/community-roadmap.md` and the onward essay outline use the new names and syntax. The `.opq` extension is kept and explained as short for *opaque* (HLD glossary, §1C).
- References to working documents under `docs/wip/` in this file and in `phase-1-analysis.md` were reworded to cite the RFC or review by number, so that the working documents can be deleted without leaving dangling paths. The documents remain in version control history.

## Re-run checks — RFC 06 (2025-12-20)

Source: RFC 06 (servers in v1). Adjudication: the RFC 06 HLD-impact review, approved in full by the user with three adjustments to keep the path open to tasks with channels:
- J-1: a request is a runtime-spawned task;
- J-2: the scratch region becomes a general task heap;
- J-3: the transport interface is versioned, with non-blocking receive reserved.

- **HLD 0.10.0.** ADR-051 is new. ADR-038, ADR-040, ADR-044 and ADR-045 are marked amended. Also added:
  - goal **G13**, not G12 as the review drafted, because G12 already exists from RFC 03;
  - the narrowed concurrency non-goal and five glossary terms;
  - the "Serve a request" sequence;
  - cross-cutting rows: shutdown, logging, back-pressure, per-task limits;
  - interface changes: the server item, the transport interface, exit codes 8 and 9, the limit keys, `InvalidServer`;
  - two risks and five changelog lines.
- **Shared types updated in every copy:**
  - `Item::Server` / `ServerDef` (5 / 4 copies);
  - `NsKind::Server` (6);
  - `DiagnosticKind::InvalidServer` (7);
  - `ProviderDecl.transport` (7);
  - `ProviderInfo.transport` with `TransportInfo` (5);
  - `RtContext.mode` value 3 where the mode values are listed (2);
  - `Limits.request_fuel` / `shutdown_grace_ms` (3);
  - exit codes 8 and 9 (2 tables).
- **`phase-1-analysis.md`:** SA-16, SA-21, SA-40 and SA-44 amended; SA-65 to SA-70 added (70 rows); five §1C terms, also in the HLD glossary (item 19).
- **`phase-2-architecture.md`:** `TransportDecl` and `RecvStatus`; §2D `limits.request_fuel` and `limits.shutdown_grace_ms`, and `limits.cpu_time_ms` limited to batch entries; D30 to D36.
- **Components:**
  - `cprime-syntax`: grammar, reserved word, E-SERVER-LINE, example, 2 scenarios;
  - `cprime-core`: tag `0x51`, namespace, configuration rows and error row;
  - `cprime-effects`: B-1, B-2, B-3, B-9 amended; B-11; error row; 5 scenarios;
  - `cprime-provider`: interface row, limits rows, step 16 (dispatch, task heaps, mode 3 unwinding, shutdown), sample transport, data model, 3 error rows, 6 scenarios, and `describe_json` gaining `"transport"`;
  - `cprime-host-providers`: all four thread-safe, with implementation rules; 2 scenarios;
  - `cprime-codegen`: step 25a, 1 scenario;
  - `cprime-cli`: root selection, server main, providers.json transport field, inventory, 3 scenarios.
- **Phase files:** `phase-4-cross-cutting.md` gains the servers section, with an error table and a scenario, and an `InvalidServer` row; `phase-5-playbook.md` gains the RFC 06 parts per step and the Phase 5 exit.
- **Deviations from the review text, recorded here:**
  - (1) Every server check, including the handler signature against the transport records, lives in `cprime-effects` B-11 rather than partly in `cprime-types`. `check_bindings` already holds the `ProviderInfo` and the binding results, so no new cross-crate interface is needed. `cprime-types` changes only in its shared-type copies.
  - (2) The recovery point is implemented by unwinding (`C-unwind` trap entry, `catch_unwind` in the worker, unwind tables in server AOT code) rather than a `setjmp`/`longjmp` shim. Rust supports unwinding through `C-unwind` frames, and the generated packages already build with `panic = "unwind"` (SA-44).
  - (3) Two playbook steps were corrected: the transport interface belongs to step 11 (SDK), and the built-ins to step 12.
- **Also fixed:**
  - `FixedBytes<32>` → `FixedBytes[32]` in the `describe_json` example (a JSON string carrying C′ type syntax, missed by the RFC 05 pass);
  - `Result<B1, B2>` → `Result[B1, B2]` in `cprime-types` C-3.
- **Checks:**
  - (1) 25 `derivedFromHld: 0.10.0` pins, none at 0.9.0, and 12 HLD files at revision 0.10.0;
  - (2) every amended spec is Draft (amended by RFC 06);
  - (3) no relative link is broken;
  - (4) no banned phrase in the RFC 06 additions, after one "handles" was reworded;
  - (5) the five new terms are in both glossaries;
  - (6) every cross-component interaction is documented on both sides:
    - `cprime_rt_serve`: `cprime-provider`, `cprime-cli`, D33;
    - `ProviderInfo.transport`: `cprime-provider` `describe_json`, `cprime-cli` D.2, `cprime-effects` B-11;
    - `cprime_handler`: `cprime-codegen` 25a, `cprime-cli` §F.
- **Part C, applied directly at the user's request on 2025-12-20:**
  - `docs/cprime-concept.md`: §2 goals and non-goals, §5.3 share-nothing counts, §6.4 `InvalidServer`, §8 Phase 5 scope and exit;
  - `README.md`: a "Servers and multi-threading" section, the scope sentence, H2, and the roadmap rows;
  - `docs/why-cprime.md`;
  - the `.cprime` TextMate grammar (`server` block, with `handler` and `transport` as keywords only inside it) and `editors/textmate/samples/api.cprime`.

**Approved.** The user promoted the 16 specs amended by RFC 06 on 2025-12-20, after this re-audit passed. The next move is `/breakdown` against the spec set, starting with the Phase 1 steps of `phase-5-playbook.md`.

## Re-run checks — RFC 07 (2025-12-20)

Source: RFC 07 (verification soundness boundaries; review points 2, 3, 4, 6, 7, 8). Adjudication: the RFC 07 HLD-impact review, approved in full by the user on 2025-12-20, including the restated criterion 8 (decision 4).

- **HLD 0.11.0.** ADR-052 to ADR-055 are new; ADR-024, ADR-025, ADR-031, ADR-033 and ADR-047 are marked amended. Also applied:
  - six glossary terms and the amended *Effect operation* row;
  - the correctness stance, the security posture (information flow and provider authority as v1 limits) and the new semantic-preservation table in `07-cross-cutting.md`;
  - interfaces, responsibilities, the golden-path and rejection sequences;
  - A-16, five corrected risk rows (including the stale AOT-leg row) and five new ones;
  - six changelog lines.
- **Shared types updated in every copy:**
  - `Severity::Note` (7 copies);
  - `DiagnosticKind::VacuousContract` and `VacuityUnknown` (7);
  - `Diagnostic.concrete` with `ConcreteCheck` (7);
  - `VerificationResult.vacuity` with `VacuityOutcome` (4);
  - `VerifyConfig.provider_wait_ms` (4);
  - `ProviderOpInfo.vectors` (5);
  - `OpDecl.vectors` (7);
  - `EffectiveEvidence` (4).
- **`phase-1-analysis.md`:** SA-14, SA-30 and SA-32 amended; SA-71 to SA-79 added (79 rows); six § 1C terms, also in the HLD glossary (item 19).
- **`phase-2-architecture.md`:** § 2C types above, `KnownAnswer`, `Lit`, reason and concreteness-reason sets; § 2D `verify.provider_wait_ms` (24 keys); § 2E `verify`/`try_verify` by value, `VerifyInput` owning the `JitModule` with `Arc` program and registry and `vacuity`, `insert_rc_opts`, `assurance::effective`, `sdk::check_vectors`; D37 to D45; the decisions heading now reads D1–D45.
- **Components:**
  - `cprime-verify`: steps 6a, 9b, 12b; steps 11 and 11b (provider thread, `fuzz_live`, evaluator context); step 12 rows; step 13 reason; data model; 11 error rows; 12 scenarios (37 in total); worked example gains `concrete`;
  - `cprime-daemon`: `--check-providers`, `assurance::effective` (§D.5), vacuity cache, `effective` reply member, `unsupported_construct` failure reason; 6 error rows; 5 scenarios;
  - `cprime-cli`: runtime build step 6a, status and inventory effective evidence, inventory `operations`; 2 error rows; 3 scenarios;
  - `cprime-provider`: `OpDecl.vectors`, `KnownAnswer`, `Lit`, the `deterministic` obligation, `check_vectors` (step 17); 1 error row; 2 scenarios;
  - `cprime-host-providers`: vector table with sources; 2 scenarios;
  - `cprime-memory`: `RcOptions`, `insert_rc_opts`; 1 scenario;
  - `cprime-codegen`: step 11a (pre-RC), Part 6 item 29, `JitModule: Send`; 2 scenarios;
  - `cprime-core`: shared types, `verify.provider_wait_ms` validation; 1 error row; 1 scenario;
  - `cprime-types`, `cprime-effects`, `cprime-syntax`: shared-type copies only.
- **Phase files:** `phase-4-cross-cutting.md` gains the new kinds, `note` severity, reasons, the `-32001` reason, the call-outcome-to-exit-code table (RFC 07 criterion 11) and three testing items; `phase-5-playbook.md` gains the RFC 07 parts per step and the Phase 4 and Phase 5 exit additions.
- **Deviations from the review text, recorded here:**
  - (1) `RtContext::sandbox` takes four arguments (`fuel`, `max_depth`, `mem_limit`, `tracker`); every new use passes `max_depth` 50,000, the T3 value.
  - (2) Known-answer byte values are cited by source and copied verbatim when the crate is written, as the existing `cprime-host-providers` examples already do; only SHA-256 of `"abc"` is spelt out, as before.
  - (3) The inventory lists vectors in a new `operations` array rather than per provider entry, because provider entries are keyed by crate and an operation count belongs to the operation.
  - (4) `assurance::effective` takes `&dyn Db`, so `cprimec` (which opens the database without a registry) can call it.
- **Pre-existing defects fixed:**
  - the fuzz thread borrowed a `!Sync` `JitModule` across threads (now moved, D41);
  - the T3 evaluator decoded the result word of a provider that had panicked (now `provider_failure`, D42);
  - the `cprime-provider` example `ProviderDecl` literal lacked `thread_safe` and `transport`;
  - `phase-4-cross-cutting.md` said 21 configuration keys after RFC 06 had made it 23 (now 24).
- **Checks:**
  - (1) 25 `derivedFromHld: 0.11.0` pins, none at 0.10.0 outside this file's history; 12 HLD files at revision 0.11.0;
  - (2) every amended spec is Draft (amended by RFC 07); `phase-3-components.md` changed only its pin;
  - (3) no relative link is broken;
  - (4) no banned phrase in the RFC 07 additions (the "process" hits name the OS process);
  - (5) all 39 † terms of § 1C are in the HLD glossary;
  - (6) every cross-component interaction is documented on both sides:
    - `assurance::effective`: `cprime-daemon` § D.5 and § G.4.h, `cprime-cli` § J and § H, § 2E;
    - `sdk::check_vectors` and `--check-providers`: `cprime-provider` step 17, `cprime-daemon` § A.2, `cprime-cli` § C.6a, § 2E;
    - `insert_rc_opts`: `cprime-memory`, `cprime-codegen` Part 6 item 29, § 2E;
    - `try_verify` by value and the vacuity cache: `cprime-verify`, `cprime-daemon` § G.4.d–f, § 2E.
- **RFC 07 criteria to spec:** 1–3 `cprime-verify` step 6a; 4 D38 and the `cprime-daemon` scenario; 5–6 D39, `cprime-daemon`, `cprime-cli`; 7 D40, `cprime-provider` step 17, `cprime-host-providers`, `cprime-cli` § C.6a; 8 D42 (restated); 9 D42; 10 D41; 11 `phase-4-cross-cutting.md` call-outcome table, HLD ADR-024 and glossary; 12–13 `cprime-verify` steps 9b and 12b; 14 D44; 15 ADR-047 amendment; 16 `cprime-codegen` Part 6 item 29; 17 HLD `07-cross-cutting.md`; 18 HLD `10-risks.md` and security posture.
- **Not in scope of the spec set:** RFC 07 Part C (`docs/cprime-concept.md` § 1, § 5.2, § 5.4, and `README.md`) belongs to `/breakdown`.

**Approved.** The user promoted the 15 specs amended by RFC 07 on 2025-12-20, after this re-audit passed. The next move is `/breakdown` of RFC 07.

## Re-run checks — ADR-056 (2025-12-20)

Source: the user's decision after RFC 07 to remove the remaining "pact" names. Applied directly at the user's request; no HLD-impact review.

- **HLD 0.12.0.** The folder moved from `docs/design/pact/` to `docs/design/cprime/` (`git mv`). ADR-056 is new; ADR-049 and ADR-050 are marked amended. One changelog line.
- **Spec set.** The spec-set id is "cprime" in all 17 spec files. SA-63 now names the new folder and id. No rule, type, interface or behaviour changed, so the specs stay **Approved**; only their pins moved, to 0.12.0.
- **Other documents.** `README.md`, `docs/cprime-concept.md` (title `C′ — AI-native compiler — concept`, next-step line), `docs/why-cprime.md`, `docs/community-roadmap.md` and the RFC 07 documents use the new path. Historical records keep the paths they were written with: `99-changelog.md` and the earlier sections of this file.
- **Not committed any more:** `docs/wip/` (RFCs, HLD-impact reviews, the essay outline) and `docs/outreach/` are in `.gitignore` and removed from the index. Their files stay on disk, and the commit history keeps their earlier versions. This file and `phase-1-analysis.md` already cite RFCs and reviews by number, so no committed link points into `docs/wip/`.
- **Checks:**
  - (1) 26 `derivedFromHld: 0.12.0` pins (25 specs plus this file); 12 HLD files at revision 0.12.0;
  - (2) no relative link in the HLD, the spec set or `README.md` is broken;
  - (3) outside the changelog, ADR-049, ADR-050, ADR-056, the two glossary history notes and the earlier sections of this file, no committed file names `docs/design/pact/` or the spec set "pact".