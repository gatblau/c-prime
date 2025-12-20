# Phase 3 — Detailed Component Specifications (index)

> **Spec set:** cprime
> **Mode:** greenfield
> **derivedFromHld:** 0.12.0
> **HLD-impact status:** adjudicated — see `phase-1-analysis.md` banner.
> **Status:** Approved (RFC 06 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit); index unchanged by RFC 07 apart from its pin
> **Last updated:** 2025-12-20

The component specs exceed the single-file size limit, so each lives in its own file under `components/`. Each file is self-contained: it copies the Phase 2 shared types it uses into its *Shared Context* and states every cross-component call it makes or serves, matching Phase 2 §2E.

| # | Component | Spec file | Roadmap phase(s) |
|---|---|---|---|
| 1 | `cprime-provider` | [components/cprime-provider.md](components/cprime-provider.md) | 2, 3 |
| 2 | `cprime-core` | [components/cprime-core.md](components/cprime-core.md) | 1, 2 |
| 3 | `cprime-syntax` | [components/cprime-syntax.md](components/cprime-syntax.md) | 1 |
| 4 | `cprime-types` | [components/cprime-types.md](components/cprime-types.md) | 1, 2 |
| 5 | `cprime-effects` | [components/cprime-effects.md](components/cprime-effects.md) | 1 |
| 6 | `cprime-memory` | [components/cprime-memory.md](components/cprime-memory.md) | 2 |
| 7 | `cprime-codegen` | [components/cprime-codegen.md](components/cprime-codegen.md) | 2, 3, 5 |
| 8 | Built-in providers | [components/cprime-host-providers.md](components/cprime-host-providers.md) | 3 |
| 9 | `cprime-verify` | [components/cprime-verify.md](components/cprime-verify.md) | 4, 5 |
| 10 | `cprime-daemon` | [components/cprime-daemon.md](components/cprime-daemon.md) | 3, 4 |
| 11 | `cprime-cli` (incl. generated runtime and release packages) | [components/cprime-cli.md](components/cprime-cli.md) | 3, 5 |
