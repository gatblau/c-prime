# Pact HLD — Changelog

Append-only. One line per edit, newest at the bottom. Format: `<date> — <origin>: <summary> → revision: <new-revision>`.

2025-12-20 — initial draft (designgen, mode=draft): bootstrapped HLD for pact from docs/cprime-concept.md revision 2 → revision: 0.1.0
2025-12-20 — direct edit: ADR-022 Perceus calling convention, inferred borrowing with owned fallback (external review b) → revision: 0.2.0
2025-12-20 — direct edit: ADR-023 acyclicity stated as a language invariant (external review c) → revision: 0.2.0
2025-12-20 — direct edit: ADR-024 termination rules, recursion only, lexicographic measures per call-graph component (external review d) → revision: 0.2.0
2025-12-20 — direct edit: ADR-025 acceptance rules and parallel obligations; T2 targets made composable (external review e, f) → revision: 0.2.0
2025-12-20 — direct edit: ADR-026 completeness rule for sum-typed results; narrowed OQ-19 to crate ownership (external review §3) → revision: 0.2.0
2025-12-20 — direct edit: ADR-027 future concurrency is share-nothing (external review §3) → revision: 0.2.0
2025-12-20 — direct edit: added IR verifier to pact-core and A-17 (external review g) → revision: 0.2.0
2025-12-20 — direct edit: replies list assumed provider operations with proved/tested results (external review a) → revision: 0.2.0
2025-12-20 — direct edit: Phase 5 exit criterion adds AOT-vs-interpreter differential testing (external review h) → revision: 0.2.0
2025-12-20 — direct edit: Motivation states the project does not depend on H1 → revision: 0.2.0
2025-12-20 — direct edit: added OQ-26 to OQ-30 and the per-phase /spec gate list (external review §4, g, h, i) → revision: 0.2.0
2025-12-20 — direct edit: fixed Mermaid syntax (quoted ER relationship labels, renamed reserved-word label "to", removed semicolon from a context edge label, renamed sequence participant alias containing "end") → revision: 0.2.1
2025-12-20 — promoted from phase-1-analysis: ADR-028 supported targets; Z3 and LLVM as pinned system installs → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: ADR-029 stdio is the only transport → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: ADR-030 per-request source synchronisation; unchanged-metadata risk recorded → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: ADR-031 acceptance header and stale implementations → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: ADR-032 project-specific pactd and pactc daemon launcher; start-up and release sequences replaced → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: ADR-033 modular verification with call-precondition obligations → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: glossary gains 13 terms; project manifest and proof obligation sharpened → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: six risks added and contract-incompleteness risk extended (vacuous requires) → revision: 0.3.0
2025-12-20 — promoted from phase-2-architecture: supporting crates row added to tech stack → revision: 0.3.0
2025-12-20 — promoted from phase-2-architecture: pact-cli crate, built-in provider crates, pact-daemon as library; diagram replaced → revision: 0.3.0
2025-12-20 — promoted from phase-2-architecture: responsibilities amended for pact-core, pact-types, pact-memory, pact-codegen, pact-provider → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: interfaces updated (pact.toml, name resolution, pactc commands, release I/O, Unit, entry impl, T3 inconclusive) → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: contract grammar additions, prelude members, call-precondition obligation kind → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: A-16 amended (evidence in acceptance header) → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: OQ-11 to OQ-19 and OQ-21 to OQ-30 resolved → revision: 0.3.0
2025-12-20 — promoted from phase-4-cross-cutting: call-depth limit added to runtime limits; pact.toml change requires restart → revision: 0.3.0
2025-12-20 — promoted from phase-1-analysis: A-3 amended by user decision (several pactd per project, write lock held only while writing, conflict check); two risks added → revision: 0.3.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-034 post-v1 directions and expressiveness parity → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-035 modules, visibility and packages → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-036 generic specs and interfaces → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-037 function values and closures → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-038 iteration, loops on evidence, tail calls post-v1 → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-039 permanent exclusions — mutable references, interior mutability, ownership annotations → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-040 share-nothing concurrency; shared memory excluded → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-041 foreign code, dynamic loading and macros excluded from .opq → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-042 primitive types, text and collections → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): ADR-043 v1 compatibility hooks → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): non-goals gain post-v1 directions; post-v1 goal P1; eight glossary terms → revision: 0.4.0
2025-12-20 — promoted from rfc-01 (/spec Branch B): three risks added (parity unmeasured, recursion depth, floating-point cost) → revision: 0.4.0
2025-12-20 — promoted from rfc-02 (/spec Branch B): ADR-040 amended — parallel map in v1, tasks next after v1 → revision: 0.5.0
2025-12-20 — promoted from rfc-02 (/spec Branch B): ADR-044 parallel map in v1 → revision: 0.5.0
2025-12-20 — promoted from rfc-02 (/spec Branch B): ADR-045 thread-safe providers → revision: 0.5.0
2025-12-20 — promoted from rfc-02 (/spec Branch B): goal G11; multi-threading non-goal reworded; four glossary terms → revision: 0.5.0
2025-12-20 — promoted from rfc-02 (/spec Branch B): responsibilities of pact-provider, pact-codegen, pact-effects amended → revision: 0.5.0
2025-12-20 — promoted from rfc-02 (/spec Branch B): configuration and runtime limits across workers → revision: 0.5.0
2025-12-20 — promoted from rfc-02 (/spec Branch B): interfaces — par_map form, limits.workers, thread_safe, InvalidParMap → revision: 0.5.0
2025-12-20 — promoted from rfc-02 (/spec Branch B): four risks added → revision: 0.5.0
2025-12-20 — direct edit: ADR-046 a person approves every contract; an agent may draft it from an agreed plan (amends ADR-001) → revision: 0.6.0
2025-12-20 — direct edit: summary, motivation, glossary, context, domain origin and interfaces reworded for contract approval; one risk added → revision: 0.6.0
2025-12-20 — promoted from rfc-03 (/spec Branch B): ADR-047 witness traces on counterexample rejections → revision: 0.7.0
2025-12-20 — promoted from rfc-03 (/spec Branch B): ADR-048 release trap locations; call chain only in opt-in builds → revision: 0.7.0
2025-12-20 — promoted from rfc-03 (/spec Branch B): goal G12; five glossary terms → revision: 0.7.0
2025-12-20 — promoted from rfc-03 (/spec Branch B): responsibilities of pact-verify, pact-codegen, pact-provider, pact-daemon, pact-cli amended → revision: 0.7.0
2025-12-20 — promoted from rfc-03 (/spec Branch B): rejection sequence gains the replay step → revision: 0.7.0
2025-12-20 — promoted from rfc-03 (/spec Branch B): tracing stance split between pactd and programs; logging and configuration rows → revision: 0.7.0
2025-12-20 — promoted from rfc-03 (/spec Branch B): observability row and A-9 amended → revision: 0.7.0
2025-12-20 — promoted from rfc-03 (/spec Branch B): interfaces — witness, release stderr location, [release] section, two keys → revision: 0.7.0
2025-12-20 — promoted from rfc-03 (/spec Branch B): four risks added → revision: 0.7.0
2025-12-20 — promoted from rfc-04 (/spec Branch B): ADR-049 the language is C′; contracts are pacts → revision: 0.8.0
2025-12-20 — promoted from rfc-04 (/spec Branch B): glossary — C′ and pact (contract) rows; Pact-IR → C′-IR → revision: 0.8.0
2025-12-20 — promoted from rfc-04 (/spec Branch B): interfaces — cprimec, cprimed, cprime_* MCP tools, cprime_provider, cprime-inventory-v1, cprime: prefix → revision: 0.8.0
2025-12-20 — promoted from rfc-04 (/spec Branch B): language and toolchain names substituted across every HLD file (ADR bodies: names only) → revision: 0.8.0
2025-12-20 — promoted from rfc-05 (/spec Branch B): ADR-050 contracts use a Go-like syntax in .cprime files; contract layer moves to the cprime stem → revision: 0.9.0
2025-12-20 — promoted from rfc-05 (/spec Branch B): ADR-043 and ADR-049 amended by ADR-050 → revision: 0.9.0
2025-12-20 — promoted from rfc-05 (/spec Branch B): glossary — Contract replaces pact (contract); statement terminator added → revision: 0.9.0
2025-12-20 — promoted from rfc-05 (/spec Branch B): interfaces — .cprime surface described in the new syntax → revision: 0.9.0
2025-12-20 — promoted from rfc-05 (/spec Branch B): risk added — readability of the new syntax unmeasured until H2 → revision: 0.9.0
2025-12-20 — promoted from rfc-05 (/spec Branch B): contract-layer names substituted across every HLD file (.cprime, .cprime/, cprime.toml, cprime.* tags, cprime-accepted) → revision: 0.9.0
2025-12-20 — direct edit (user decision after rfc-05): glossary — `.opq` explained as short for opaque; the extension is kept → revision: 0.9.0
2025-12-20 — promoted from rfc-06 (/spec Branch B): ADR-051 servers in v1 — each request is a runtime-spawned share-nothing task (adjusted at adjudication to keep the path to tasks with channels) → revision: 0.10.0
2025-12-20 — promoted from rfc-06 (/spec Branch B): ADR-038, ADR-040, ADR-044 and ADR-045 amended by ADR-051 → revision: 0.10.0
2025-12-20 — promoted from rfc-06 (/spec Branch B): goal G13 added; concurrency non-goal narrowed; glossary — server, handler, transport, task, task heap → revision: 0.10.0
2025-12-20 — promoted from rfc-06 (/spec Branch B): architecture responsibilities; sequence "Serve a request"; cross-cutting shutdown, logging, back-pressure and per-task limits → revision: 0.10.0
2025-12-20 — promoted from rfc-06 (/spec Branch B): interfaces — server item, versioned transport interface, exit codes 8 and 9, limit keys; two risks added → revision: 0.10.0
2025-12-20 — promoted from rfc-07 (/spec Branch B): ADR-052 vacuous and inconsistent contracts rejected before any obligation → revision: 0.11.0
2025-12-20 — promoted from rfc-07 (/spec Branch B): ADR-053 effective evidence computed at report time, never persisted → revision: 0.11.0
2025-12-20 — promoted from rfc-07 (/spec Branch B): ADR-054 deterministic means effect-free; known-answer vectors; bounded provider calls during checking → revision: 0.11.0
2025-12-20 — promoted from rfc-07 (/spec Branch B): ADR-055 counterexamples classified concrete or abstract; ADR-024, ADR-025, ADR-031, ADR-033 and ADR-047 amended → revision: 0.11.0
2025-12-20 — promoted from rfc-07 (/spec Branch B): glossary — six terms, effect operation amended; correctness and security posture; semantic-preservation assumptions table → revision: 0.11.0
2025-12-20 — promoted from rfc-07 (/spec Branch B): interfaces, responsibilities and two sequences; risks corrected (vacuity, AOT leg) and five added; A-16 amended → revision: 0.11.0
2025-12-20 — direct edit (user decision after rfc-07): ADR-056 internal topic ids use the cprime stem — HLD folder docs/design/cprime/, spec-set id "cprime"; ADR-049 and ADR-050 amended → revision: 0.12.0
