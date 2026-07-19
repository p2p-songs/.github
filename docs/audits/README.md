# Audit Registry

This is the authoritative entry point for adversarial-review status. Read the
first row before starting implementation; it is the latest audit. Older reports
remain as history and may contain findings that were subsequently resolved.

| ID | Date | Scope | Status | Supersedes | Open findings |
|---|---|---|---|---|---|
| **A-005** | 2026-07-19 | [Addon SDK implementation](./2026-07-19-addon-sdk-implementation.md) | **CHANGES REQUIRED — 2 critical, 3 medium** | A-004 for current implementation sign-off | 5 pending publication/triage |
| **A-004** | 2026-07-19 | [Protocol implementation](./2026-07-19-protocol-implementation.md) | **RECONCILED — all 4 findings (3 medium, 1 low) addressed 2026-07-19; re-audit to confirm** | A-003 for current implementation sign-off | None pending re-audit |
| **A-003** | 2026-07-17 | [Product-wide plan](./2026-07-17-product-wide-plan.md) | **RESOLVED — the 1 high reconciled 2026-07-18; plan signed off for the declared scope** | A-001 and A-002 for current plan sign-off | None ([addon-sdk#1](https://github.com/p2p-songs/addon-sdk/issues/1) closed) |
| A-002 | 2026-07-17 | [Core-player architecture re-audit](./2026-07-17-core-player-architecture-reaudit.md) | Resolved; historical | A-001 for core architecture | None |
| A-001 | 2026-07-17 | [Core-player plan](./2026-07-17-core-player-plan.md) | Resolved; historical | — | None |

## Current decision

A-005 confirms all four A-004 protocol fixes, then audits Phase 2's
`@p2p-songs/addon-sdk`. **Current implementation does not pass: 2 critical and
3 medium findings.** Configured manifest/resource URLs carry credentials but
are eligible for public caching; handler/router exception details are returned
verbatim and can disclose those credentials; stream/lyrics accept non-track
types; `configurationRequired` fails open; malformed percent encoding escapes
the router's response boundary. A-003 remains the resolved plan audit.

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
