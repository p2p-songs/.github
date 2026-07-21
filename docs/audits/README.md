# Audit Registry

This is the authoritative entry point for adversarial-review status. Read the
first row before starting implementation; it is the latest audit. Older reports
remain as history and may contain findings that were subsequently resolved.

| ID | Date | Scope | Status | Supersedes | Open findings |
|---|---|---|---|---|---|
| **A-010** | 2026-07-21 | [Player P-5 minimal app](./2026-07-21-player-p5.md) | **CHANGES REQUIRED — 1 medium** | A-009 for current implementation sign-off | 1 |
| **A-009** | 2026-07-21 | [Player P-4 persistence and catalog fan-out](./2026-07-21-player-p4.md) | **RECONCILED — all 3 medium addressed 2026-07-21; re-audit to confirm** | A-008 for current implementation sign-off | None pending re-audit |
| **A-008** | 2026-07-21 | [Player P-3 addon client](./2026-07-21-player-p3.md) | **RECONCILED — both (2 medium) addressed 2026-07-21; re-audit to confirm** | A-007 for current implementation sign-off | None pending re-audit |
| **A-007** | 2026-07-20 | [Player P-1 and A-006 reconciliation](./2026-07-20-player-p1.md) | **RECONCILED — both (1 high, 1 medium) addressed 2026-07-20; re-audit to confirm** | A-006 for current implementation sign-off | None pending re-audit |
| **A-006** | 2026-07-19 | [Reference addons and SDK re-audit](./2026-07-19-reference-addons.md) | **RECONCILED — all 6 (1 critical, 5 medium) addressed 2026-07-20; re-audit to confirm** | A-005 for current implementation sign-off | None pending re-audit |
| **A-005** | 2026-07-19 | [Addon SDK implementation](./2026-07-19-addon-sdk-implementation.md) | **RECONCILED — all 5 (2 critical, 3 medium) addressed 2026-07-19; re-audit to confirm** | A-004 for current implementation sign-off | None pending re-audit |
| **A-004** | 2026-07-19 | [Protocol implementation](./2026-07-19-protocol-implementation.md) | **RECONCILED — all 4 findings (3 medium, 1 low) addressed 2026-07-19; re-audit to confirm** | A-003 for current implementation sign-off | None pending re-audit |
| **A-003** | 2026-07-17 | [Product-wide plan](./2026-07-17-product-wide-plan.md) | **RESOLVED — the 1 high reconciled 2026-07-18; plan signed off for the declared scope** | A-001 and A-002 for current plan sign-off | None ([addon-sdk#1](https://github.com/p2p-songs/addon-sdk/issues/1) closed) |
| A-002 | 2026-07-17 | [Core-player architecture re-audit](./2026-07-17-core-player-architecture-reaudit.md) | Resolved; historical | A-001 for core architecture | None |
| A-001 | 2026-07-17 | [Core-player plan](./2026-07-17-core-player-plan.md) | Resolved; historical | — | None |

## Current decision

A-010 audited the P-5 minimal player application and rechecked the active
cross-repo invariants. **One medium change is required:** debounced queue
persistence does not flush on close/reload, so recent queue edits can be lost.
After scope clarification, the initial CSP/Trusted Types concern was withdrawn
as premature until credential-bearing addons land, and mobile layout was
withdrawn as deferred polish outside this lite desktop E2E slice. All automated
suites, typechecks, and builds pass. See the A-010 report for evidence and the
six-lens disposition.

A-009 audited player P-4 and rechecked A-008, finding 3 medium. **All three are
now reconciled (2026-07-21):** `askBounded`'s deadline is a **hard** bound — it
races the task against a timer it controls, so a transport that ignores its abort
signal can no longer wedge a fan-out (the doc comment that overclaimed a
"per-call deadline" is corrected too); the read-modify-write atomicity primitive
now lives in the **port** (`PersistenceStore.update`, a Dexie `rw` transaction /
a synchronous memory section) with all three RMW sites moved onto it, so
overlapping playlist edits no longer silently discard one another; and the
explicitly-scoped **play history** is implemented as an identity-only collection
with a retention cap, with the Dexie schema declared cumulatively (v1→v2) so
existing databases upgrade. 166 player tests and built-output probes green — each
probe reproducing the auditor's own. The reconciliation is in the A-009 report's
Resolution section. **A re-audit is invited to confirm.** A-008's isolation work
is confirmed and now genuinely bounded; A-003 remains the resolved plan audit.

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
