# C′ — Regulatory and Compliance Context

> **Mode:** draft
> **Generated from:** inline context — `docs/cprime-concept.md` (Draft, revision 2)
> **Revision:** 0.12.0
> **Last updated:** 2025-12-20

**N/A — non-regulated.** C′ is a local developer tool and compiler. It processes no personal data, holds no customer records, and operates in no regulated sector. The concept names no standard, jurisdiction or statutory obligation. Programs built *with* C′ may be subject to regulation; that is the obligation of the project that builds and deploys them, not of the compiler.

One non-statutory obligation is recorded because the design affects it:

| Obligation | Source | How the design addresses it |
|---|---|---|
| Licence compliance for code statically linked into release binaries (Cranelift, LLVM runtime parts, Z3 if linked, provider crates) | Licences of the third-party dependencies | The provider inventory lists every linked provider crate and version per release build. Licence review of the compiler's own dependencies and of what reaches release binaries is OQ-23 (non-blocking). |
