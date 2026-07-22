# Audit Registry

This is the authoritative entry point for adversarial-review status. Read the
first row before starting implementation; it is the latest audit. Older reports
remain as history and may contain findings that were subsequently resolved.

| ID | Date | Scope | Status | Supersedes | Open findings |
|---|---|---|---|---|---|
| **A-012** | 2026-07-22 | [Product implementation re-audit](./2026-07-22-product-reaudit.md) | **RECONCILED — the 1 critical addressed 2026-07-22; re-audit to confirm** | A-011 for current implementation sign-off | None pending re-audit |
| **A-011** | 2026-07-21 | [Bitbop and browser security](./2026-07-21-bitbop-and-browser-security.md) | **RECONCILED — all 3 (1 critical, 2 medium) addressed 2026-07-21; re-audit to confirm** | A-010 for current implementation sign-off | None pending re-audit |
| **A-010** | 2026-07-21 | [Player P-5 minimal app](./2026-07-21-player-p5.md) | **RECONCILED — the 1 medium addressed 2026-07-21; re-audit to confirm** | A-009 for current implementation sign-off | None pending re-audit |
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

A-012 re-audited the implemented product and the A-011 reconciliation. It
found **1 critical**: Bitbop's IPv6 policy recognizes IPv4-mapped addresses only
when the embedded IPv4 is dotted decimal. Hexadecimal forms such as
`::ffff:7f00:1` (IPv4 loopback) and `::ffff:a9fe:a9fe` (link-local/cloud
metadata) fell through as public and were passed directly to the socket, so
public-safe mode was still an SSRF proxy for HTTPS-speaking internal targets.
**Reconciled 2026-07-22:** the policy now parses IPv6 into eight words and
judges any embedded IPv4 numerically, so every spelling of one address gets one
answer. The deeper lesson is recorded in the checklist — `new URL()` rewrites
`[::ffff:127.0.0.1]` to `[::ffff:7f00:1]`, so the dotted form the old regex
caught, and that A-011's test asserted, was the one spelling that could never
reach the check: a passing test that defended nothing. See the A-012 report for
the direct probe and six-lens disposition.

A-011 audited the newly landed Bitbop debrid addon and player browser-security
gate, finding 1 critical + 2 medium. **All three are now reconciled
(2026-07-21).** The critical was a real miss: Bitbop fetches a caller-supplied
Torznab URL server-side and nothing policed the destination. Indexer requests
now go through a guarded transport that enforces scheme policy, **re-validates
every redirect hop** (a permitted public URL 302-ing to loopback defeats a
pre-flight-only check), and **connects to the address it validated** via
`node:http`'s `lookup` hook, leaving no DNS-rebinding window. Writing the
regression test surfaced a further hole the original probe would have missed —
a **literal IP host never triggers DNS**, so the hook never fired and
`https://169.254.169.254/…` passed; literal hosts (incl. bracketed IPv6 and
`::ffff:` IPv4-mapped forms) are now checked separately. Public-safe is the
**default**; self-hosters, whose Jackett is typically on loopback, opt in with
`BITBOP_ALLOW_PRIVATE_INDEXERS=1`. A total debrid outage is now distinguished
from a legitimate empty answer and returns a retryable uncacheable error, and
the two nonfunctional configure options were removed from the **schema** as well
as the page so an unusable install URL can no longer be produced or parsed.
Verified by reproducing the auditor's probe against a listener that records
connections: blocked with zero contact in public mode, reachable in self-host
mode, no key material in diagnostics. `addons` 168 tests (Bitbop 66 → 122). See
the A-011 report's Resolution section.

A-010 audited the P-5 minimal player application and rechecked the active
cross-repo invariants, finding 1 medium: debounced queue persistence did not
flush on close/reload, so recent queue edits could be lost. **It is now
reconciled (2026-07-21)** — and the finding turned out to understate the defect.
The debounce was keyed on *any* engine notification, and the engine notifies
~4×/s on position ticks, so the trailing write was reset faster than it could
fire: nothing was persisted for the entire duration of playback, not merely
within an 800 ms window. The debounce moved into a DOM-free `SessionAutosave`
that reschedules **only on a changed queue snapshot**, **flushes** the pending
snapshot on `visibilitychange`→hidden / `pagehide` / teardown, and keeps a
rejected write pending for the next flush without spinning. After scope
clarification, the initial CSP/Trusted Types concern was withdrawn as premature
until credential-bearing addons land, and mobile layout was withdrawn as
deferred polish outside this lite desktop E2E slice. All automated suites,
typechecks, and builds pass (player now 180 tests). See the A-010 report for
evidence and the six-lens disposition.

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
