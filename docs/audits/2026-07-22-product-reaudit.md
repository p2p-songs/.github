# Product implementation re-audit

- **Audit ID:** A-012
- **Status:** RECONCILED — the 1 critical addressed 2026-07-22; re-audit to confirm
- **Supersedes:** A-011 for current implementation sign-off
- **Audited commits:** `.github` `dc43e1e8de208939824bef18613a6586bad2a5ee`; `player` `a8f54240964beebd2b18ca57c06f61b9f8dd256f`; `addon-sdk` `b9a55a7e7ea0a324da92f49ee00df70dd6e0c9c2`; `addons` `9cf045c4b0c54badbd2bb2c3c89dff2a62088e12`; `backend` `682adc7ed6b5db10d37db9b8a344b65b663e17f9`
- **Last updated:** 2026-07-22

## Scope and verdict

This pass re-audited the implemented product across all six required lenses,
with emphasis on the A-011 reconciliation, configured-URL secrecy, protocol
boundaries, Real-Debrid account mutation, player persistence and stale async
completion handling, and the implemented first-launch/search/play/recovery
journeys. **Changes are required.** The A-011 SSRF fix is incomplete: its IPv6
classifier recognizes IPv4-mapped addresses only when the embedded IPv4 is
written in dotted decimal. Equivalent hexadecimal forms are accepted as public
and are passed directly to the socket, permitting access to loopback,
link-local, and private IPv4 targets through an IPv6 literal. No additional
finding survived the checklist's evidence and present-scope calibration gates.
The optional backend, PWA, `stream-ytmusic`, catalog charts, and lyrics remain
declared future/scaffolding scope and were not credited as implemented.

## Findings

### [CRITICAL] Hexadecimal IPv4-mapped IPv6 literals bypass the SSRF address policy

- **Category:** technical-soundness
- **Repo / file / line:** `addons/packages/bitbop/src/net/ip-policy.ts:54-68`; `addons/packages/bitbop/src/net/guarded-fetch.ts:100-106`
- **Reference:** Checklist §3, “The caller-supplied indexer URL is policed before it is fetched” (audit A-011), especially literal IP and IPv4-mapped IPv6 handling
- **Finding:** `isPublicIpv6()` recognizes an IPv4-mapped/compatible address only when its embedded IPv4 is dotted decimal (`::ffff:127.0.0.1`). Standard hexadecimal representations such as `::ffff:7f00:1` and `::ffff:a9fe:a9fe` do not match that regular expression and fall through to `true`. `URL` preserves these canonical hexadecimal host forms, `isIP()` reports family 6, and the literal-host path therefore accepts them without DNS lookup.
- **Why it matters:** In public-safe mode, an arbitrary caller can configure an HTTPS indexer such as `https://[::ffff:7f00:1]/…` and make Bitbop connect to IPv4 loopback (`127.0.0.1`) through the IPv6-mapped address. The same encoding reaches link-local/cloud-metadata (`169.254.169.254`) and private IPv4 ranges. This restores the server-side request-forgery capability A-011 was intended to close; the HTTPS-only rule reduces the set of reachable internal services but does not restore the promised destination boundary.
- **Suggested fix:** Parse IPv6 numerically and classify mapped/compatible embedded IPv4 bits independent of textual representation. Prefer a well-tested IP/CIDR parser over prefix regular expressions, and add literal-host plus redirect tests for hexadecimal mapped loopback, link-local, RFC1918, and a genuinely public mapped address.
- **Verdict:** CONFIRMED
- **Reconciled:** 2026-07-22, `addons` `ip-policy.ts`. `isPublicIpv6()` now parses
  the literal into eight 16-bit words and classifies numerically: mapped
  (`::ffff:0:0/96`), translated (`::ffff:0:0:0/96`) and compatible (`::/96`)
  forms are all judged by their embedded IPv4, and the remaining ranges are
  matched by masked word comparison rather than string prefix. Unparseable input
  denies. `ipv4Octets()` was tightened to plain decimal, since `Number()` alone
  read `0x7f` as 127. Tests added for hex mapped loopback / metadata / RFC1918,
  the fully expanded form, translated and compatible loopback, a genuinely
  public mapped address, and malformed input — plus four `guarded-fetch`
  literal-host cases asserting `non-public address` (not a connection failure).
  **The audit understated it in one respect worth recording:** `new URL()`
  *rewrites* `[::ffff:127.0.0.1]` to `[::ffff:7f00:1]`, so the dotted form the
  old regex recognized — and that the A-011 test asserted — is the one spelling
  that could never reach the check. The passing test was vacuous with respect to
  the path it was defending.

## Six-lens disposition

- **Technical soundness:** changes required for the confirmed SSRF bypass. Protocol schemas/router validation, configured-response no-store behavior, player command/query-plane separation, identity-stamped async commits, JIT resolution, and resolved-media non-persistence otherwise passed inspection and tests.
- **Legal validity:** no additional finding. Bitbop remains one self-contained addon, uses the requesting user's credentials, ships no tracker list, stores no audio, selects a specific track file, and returns resolved HTTPS URLs. `stream-legal` remains a fixed, per-item-license-checked source set.
- **Overall implementation quality:** one critical hostile-input gap. All executable suites, typechecks, and production builds passed after allowing their local probe servers; the missing test case is precisely the alternate hexadecimal representation that bypasses the regex.
- **System design and operations:** no additional finding. Provider outage/no-match handling, bounded fan-out, Real-Debrid probe cleanup, non-mutating pre-check, settle budget, and provider backoff are represented in executable code and regression tests. The backend is scaffolding-only and was not audited as a deployed sync service.
- **UI/UX and accessibility:** no finding for present scope. The implemented UI exposes addon installation, redacts configured URLs, surfaces playback failure, and uses native labeled controls in the primary paths. Mobile, PWA, source selection, and the full theme contract remain explicitly incomplete rather than silently claimed.
- **End-to-end user value and delight:** no additional finding for the implemented slice. The real HTTP fixture path and recorded operator smoke establish install/search/resolve/play composition; queue persistence and error recovery are covered. The incomplete addon/PWA/sync features constrain breadth but are openly documented future scope.

## Verification

- Direct executable probe: `isPublicAddress("::ffff:7f00:1", 6)`, `isPublicAddress("::ffff:a9fe:a9fe", 6)`, and `isPublicAddress("0:0:0:0:0:ffff:7f00:1", 6)` all returned `true` in the audited build. `new URL("https://[0:0:0:0:0:ffff:7f00:1]/").hostname` canonicalized to `[::ffff:7f00:1]`, confirming the value reaches the literal-host classifier in bypass form.
- `player`: 198/198 tests passed; typecheck and Vite production build passed.
- `addon-sdk`: 46 protocol tests and 36 SDK tests passed; typechecks and builds passed.
- `addons`: 27 MusicBrainz, 17 musicmeta, 25 stream-legal, and 160 Bitbop tests passed; typechecks and builds passed.
- Local-listener tests initially failed under filesystem/network sandboxing (`listen EPERM`) and passed unchanged when rerun with loopback binding permission; this was an environment restriction, not a product failure.
