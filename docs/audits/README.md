# Audit Registry

This is the authoritative entry point for adversarial-review status. Read the
first row before starting implementation; it is the latest audit. Older reports
remain as history and may contain findings that were subsequently resolved.

| ID | Date | Scope | Status | Supersedes | Open findings |
|---|---|---|---|---|---|
| **A-004** | 2026-07-19 | [Protocol implementation](./2026-07-19-protocol-implementation.md) | **OPEN — changes required (3 medium, 1 low)** | A-003 for current implementation sign-off | 3 medium, 1 low (report only) |
| **A-003** | 2026-07-17 | [Product-wide plan](./2026-07-17-product-wide-plan.md) | **RESOLVED — the 1 high reconciled 2026-07-18; plan signed off for the declared scope** | A-001 and A-002 for current plan sign-off | None ([addon-sdk#1](https://github.com/p2p-songs/addon-sdk/issues/1) closed) |
| A-002 | 2026-07-17 | [Core-player architecture re-audit](./2026-07-17-core-player-architecture-reaudit.md) | Resolved; historical | A-001 for core architecture | None |
| A-001 | 2026-07-17 | [Core-player plan](./2026-07-17-core-player-plan.md) | Resolved; historical | — | None |

## Current decision

A-004 audits the first implementation slice, `@p2p-songs/protocol`, and requires
changes: direct resource URLs accept arbitrary schemes, metadata type/identity
pairings are unenforced while playlist identity is undefined, the Phase 1
standalone protocol specification is absent, and the required bonus-disc test
fixture is missing. **Implementation does not currently pass.** A-003 remains
the resolved plan audit; its entity-typed recording/track decision is correctly
represented in the implementation audited here.

## Reading rules

- Newest audit is the first table row, identified by the highest audit ID.
- `OPEN` means at least one finding still blocks the audit's sign-off.
- `RESOLVED` means its findings were addressed; the report remains evidence,
  not the current verdict.
- `SUPERSEDED` means a later audit owns current sign-off for that scope.
- GitHub issues track actionable high/critical findings, but this registry—not
  the issue list—is the authoritative sequence and overall verdict.
- Every new report must declare `Audit ID`, `Status`, `Supersedes`, `Audited
  commits`, and `Last updated` directly below its title, then update this table.
