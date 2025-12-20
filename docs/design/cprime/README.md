# C′ — High-Level Design

> **Mode:** draft
> **Generated from:** inline context — `docs/cprime-concept.md` (Draft, revision 2)
> **Revision:** 0.12.0
> **Last updated:** 2025-12-20

## Orientation

This is the high-level design for C′ (`cprimec`/`cprimed`), a verification-first systems compiler written in Rust. Humans write `.cprime` contracts or approve ones an agent drafted from a plan, AI agents write `.opq` implementations, and a long-running daemon accepts an implementation only after it type-checks, passes the effect check, terminates, and meets its contract at the evidence level the spec demands. The design covers system context, the ten-crate architecture with four built-in provider crates, the principal sequences (implement a spec, rejection, fuzzing fallback, start-up, release build), the domain model, tech stack, cross-cutting stances, external interfaces, 21 ADRs carried from the concept plus six (ADR-022 to ADR-027) added after an external review on 2025-12-20, six more (ADR-028 to ADR-033) promoted from `/spec` on 2025-12-20, ten more (ADR-034 to ADR-043) recording post-v1 directions from RFC 01, two (ADR-044, ADR-045) adding parallel map from RFC 02, one (ADR-046) letting agents draft contracts for a person to approve, two (ADR-047, ADR-048) adding witness traces and release trap locations from RFC 03, one (ADR-049) renaming the language to C′, one (ADR-050) giving contracts a Go-like syntax in `.cprime` files, one (ADR-051) letting a release binary serve requests, each in a runtime-spawned share-nothing task, and four (ADR-052 to ADR-055) bounding what verification establishes: vacuous contracts, effective evidence across callees, the meaning and checking of `deterministic` operations, and concrete versus abstract counterexamples, and one (ADR-056) moving the internal topic ids to the `cprime` stem; it also covers the phased roadmap. It is for the engineers who will run `/spec` against it to produce the low-level spec set. All open questions are non-blocking; only OQ-20 is open.

## Contents

- [00 — Overview](./00-overview.md) — Summary, Motivation, Goals, Non-goals, Glossary.
- [01 — Context](./01-context.md) — System context diagram, external actors, neighbouring systems.
- [02 — Architecture](./02-architecture.md) — Component diagram, tier pipeline, crate responsibilities.
- [03 — Sequences](./03-sequences.md) — Golden path, rejection, fuzzing fallback, start-up, release build.
- [04 — Domain](./04-domain.md) — Domain Model, actions, Data Lifecycle.
- [05 — Tech stack](./05-tech-stack.md) — Tech choices and rationale.
- [06 — Regulatory](./06-regulatory.md) — N/A — non-regulated; licence note.
- [07 — Cross-cutting](./07-cross-cutting.md) — Cross-cutting Concerns table, NFRs.
- [08 — Interfaces](./08-interfaces.md) — JSON-RPC, MCP, file formats, SDK, CLI, configuration (names and shapes only).
- [09 — Decisions](./09-decisions.md) — ADR-001 to ADR-056.
- [10 — Risks](./10-risks.md) — Assumptions, Open Questions, Risks, Alternatives, Dependencies, Rollout/Rollback.
- [99 — Changelog](./99-changelog.md) — HLD revision history.
