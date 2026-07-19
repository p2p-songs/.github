# Protocol Implementation Audit

- **Audit ID:** A-004
- **Status:** RECONCILED — all 4 findings addressed 2026-07-19 (see Resolution); re-audit invited to confirm
- **Supersedes:** A-003 for current implementation sign-off; A-003 remains the resolved plan audit
- **Audited commits:** `.github` `3fbce816394e9cb20b7ab6ff88d4eeec7e127f3b`; `addon-sdk` `7eb05963e523f5e854a561ca6e63cc727ee2def3`; `player` `af63ee23746ff6d601aa420439bed5540845c6dd`; `addons` `459cb5bb5017ae69d29f7e5c5b948e62e51b8493`; `backend` `682adc7ed6b5db10d37db9b8a344b65b663e17f9`
- **Last updated:** 2026-07-19

## Scope

This pass audits the first implemented vertical slice, `addon-sdk/packages/protocol`, against Implementation Plan Phase 1 and §8, Review Checklist §1, §6, and §7, and the repo-specific invariants. The `player`, `addons`, and `backend` repositories contain scaffolding/docs only, so their later-phase implementation invariants and full product journeys are not treated as implemented promises in this pass. Their committed plans were checked for composition with the protocol surface. The audit included source and package inspection, all 26 tests, typecheck, build, runtime schema probes, repository history/status, and the current open issue/PR state.

## Overall verdict

The implemented foundation gets the central A-003 decision right: MBIDs are entity-typed, stream and lyrics requests require recording identity, optional album context is represented, source shapes are mutually exclusive, and expiry hints are optional and mutually exclusive. The test, typecheck, and build suites pass. It is not ready to serve as the canonical wire contract, however: direct-resource URL schemas accept non-HTTP and executable schemes despite the HTTPS promise; metadata permits contradictory type/identity pairs while `playlist` has no defined valid identity; and the required standalone protocol specification is absent. One required A-003 fixture, the bonus-disc case, is also not actually tested. These are bounded protocol-layer defects, not evidence of legal drift in the unimplemented addons.

## Findings

### [MEDIUM] Direct resource schemas accept arbitrary URL schemes

- **Category:** technical-soundness
- **Repo / file / line:** `addon-sdk/packages/protocol/src/stream.ts:44`; `addon-sdk/packages/protocol/src/lyrics.ts:9`; `addon-sdk/packages/protocol/src/meta.ts:16`; `addon-sdk/packages/protocol/src/manifest.ts:54-55`
- **Reference:** Plan opening promise and §8 (resolved direct, Range-servable HTTPS links)
- **Finding:** The wire schemas use `z.string().url()`, which validates URL syntax but does not restrict the scheme. Runtime probes confirmed that a stream object accepts `ftp://x/y`, `javascript:alert(1)`, and `data:audio/mpeg;base64,AA==`; plain `http://` is accepted as well. The same unconstrained primitive is used for lyrics, artwork, logo, and background URLs.
- **Why it matters:** An untrusted addon can return a value that passes the canonical validator but cannot satisfy browser playback/fetch requirements, bypasses the protocol's secure-transport promise, or reaches a scheme that a future UI consumer must remember to reject independently. This turns the supposed runtime trust boundary into syntax-only validation and makes behavior depend on how each consumer binds the URL.
- **Suggested fix:** Define and reuse resource-specific URL schemas. Require `https:` for remote media and fetched resources; explicitly document and narrowly allow any deliberate development exception rather than accepting every URL scheme. Add negative tests for `http`, `ftp`, `file`, `data`, and `javascript` as appropriate to each field.
- **Verdict:** CONFIRMED

### [MEDIUM] Content type and identity can contradict each other, and playlist identity is undefined

- **Category:** technical-soundness
- **Repo / file / line:** `addon-sdk/packages/protocol/src/meta.ts:10-19`; `addon-sdk/packages/protocol/src/ids.ts:23-34,67-69`
- **Reference:** Plan §5 and §8; Checklist §6 content-type and entity-typed-ID items
- **Finding:** `metaPreviewSchema` independently combines `type: artist|album|track|playlist` with `id: any MBID|ISRC`. Runtime validation therefore accepts an `artist` whose id is `mbid:recording:...`, an `album` whose id is a track, and similar contradictions. Separately, the protocol advertises `playlist` as a content type but defines no playlist ID namespace: MusicBrainz supplies none of the four accepted MBID entities for a playlist, and ISRC identifies recordings, not playlists.
- **Why it matters:** Addons can emit schema-valid metadata that routes subsequent `/meta`, queue, cache, and dedup operations under the wrong entity semantics. A conforming third-party addon cannot assign an honest, stable ID to a playlist at all, so one advertised protocol branch is unusable without inventing an ad hoc identifier that the canonical schema rejects (or lying with another entity's ID).
- **Suggested fix:** Make metadata a discriminated union (at minimum artist→artist MBID, album→release MBID, track→recording MBID/ISRC, with album-context track IDs represented separately). Decide and document a neutral playlist ID namespace, then add its schema and round-trip tests; otherwise remove `playlist` until that contract exists.
- **Verdict:** CONFIRMED

### [MEDIUM] The required standalone wire specification is missing

- **Category:** system-design
- **Repo / file / line:** `addon-sdk/docs/PROTOCOL.md` (absent); implemented schemas under `addon-sdk/packages/protocol/src/`
- **Reference:** Plan §10, Phase 1
- **Finding:** Phase 1 explicitly requires `docs/PROTOCOL.md` covering the manifest schema, resources/types, ID scheme, and stream/lyrics shapes, but the repository has no `docs` directory or protocol document. The package README points readers back to the broad implementation plan rather than a versioned route-and-payload specification.
- **Why it matters:** The product promise is an independently implementable HTTP+JSON addon protocol. Zod source files describe object fragments but do not define HTTP routes, encoding, status/error behavior, or a stable third-party-facing contract. Phase 2's router and external addon authors would otherwise have to reverse-engineer implementation code, increasing incompatible interpretations at the primary system boundary.
- **Suggested fix:** Add the Phase 1 `docs/PROTOCOL.md`, version it as v0.1, include complete request/response examples and the hand-written manifest plus `/stream` validation required by the exit criteria, and reconcile it with the final schemas.
- **Verdict:** CONFIRMED

### [LOW] The required bonus-disc identity fixture is not tested

- **Category:** implementation-quality
- **Repo / file / line:** `addon-sdk/packages/protocol/test/ids.test.ts:59-92`; `addon-sdk/packages/protocol/test/stream.test.ts:85-97`
- **Reference:** Plan §8 identity fixtures; Checklist §6 identity-fixture item
- **Finding:** Tests cover distinct track IDs, a free-text `A4` position, rejection of the old synthetic form, and the same recording on two releases. No test models or names the required bonus-disc fixture. The multi-disc assertion only compares two standalone constants and does not exercise album-track objects with distinct disc/position context.
- **Why it matters:** The checklist names bonus discs separately because they stress release ordering/context beyond the minimal two-ID distinction. Reporting all A-003 fixtures as covered is not supported by the test suite, and later changes could drop disc/context behavior without this required regression example failing.
- **Suggested fix:** Add an explicit bonus-disc album-track fixture (and preferably a full multi-disc album fixture) asserting stable recording identity, distinct track identity, disc number, and free-text/position preservation.
- **Verdict:** CONFIRMED

## Six-lens summary

- **Technical soundness:** Changes required; arbitrary URL schemes and inconsistent/undefined content identity are confirmed above.
- **Legal validity:** No finding in the implemented scope. No stream addon, debrid integration, source catalog, credentials, or media persistence exists yet; legal invariants must be re-audited when those components land.
- **Overall implementation quality:** The package is compact, schema-first, strictly typed, and green on 26 tests/typecheck/build. The explicit bonus-disc regression fixture is missing.
- **System design and operations:** Changes required because the third-party wire specification and Phase 1 route-level exit artifact are absent. Deployment, migration, concurrency, retry, and observability concerns are not yet applicable to this schema-only slice.
- **UI/UX and accessibility:** No finding in scope; there is no implemented UI. The protocol defects would surface as confusing playback/identity failures later, but no unimplemented UI requirement is being failed here.
- **End-to-end user value and delight:** The full first-launch/playback/recovery journeys cannot yet run because later phases are scaffolding. Within the slice, clean installation, tests, typecheck, and build were verified; no claim of end-to-end product readiness is made.

## Verification record

- `pnpm test`: 3 files, 26 tests passed.
- `pnpm typecheck`: passed.
- `pnpm build`: passed.
- Built-package runtime probes confirmed acceptance of `http:`, `ftp:`, `javascript:`, and `data:` stream URLs and acceptance of an `artist` metadata object carrying a recording MBID.
- `gh issue list` and `gh pr list` for `p2p-songs/addon-sdk`: no open entries returned at audit time.
- No code in `player`, `addon-sdk`, `addons`, or `backend` was modified by this audit.

## Resolution (2026-07-19, implementer)

All four findings addressed in `@p2p-songs/protocol`; docs synced (Plan §8,
Review Checklist §6 + Current status, addon-sdk `CLAUDE.md`).

- **[MEDIUM] Arbitrary URL schemes → FIXED.** New `src/url.ts` exports a shared
  `httpsUrlSchema` (`z.string().url()` + an `^https://` scheme refine). Applied
  to `stream.url`, `lyric.url`, `poster`, `logo`, and `background`. `ytId` /
  `infoHash` are not URLs and remain exempt. `test/url.test.ts` adds negative
  cases for `http`/`ftp`/`file`/`data`/`javascript`, and a built-package runtime
  probe confirms the four A-004 attack strings are now rejected.
- **[MEDIUM] Type/identity contradictions + undefined playlist identity → FIXED.**
  `metaPreviewSchema` and `metaDetailSchema` are now `z.discriminatedUnion("type", …)`:
  `artist`→artist MBID, `album`→release MBID, `track`→recording MBID/ISRC,
  `playlist`→a new `playlist:<token>` namespace (addon-scoped opaque token, no
  colon; branded `playlistIdSchema`, `parseId` support). `test/meta.test.ts`
  asserts each honest pairing is accepted and each contradiction rejected.
- **[MEDIUM] Missing standalone spec → FIXED.** `addon-sdk/docs/PROTOCOL.md`
  (v0.1) now defines transport rules, content types, the ID scheme incl.
  playlist, HTTP routes, per-resource request/response payloads, empty-vs-error
  behavior, `/configure`, versioning, and a worked hand-written manifest +
  `/stream` validation example. The package README points to it.
- **[LOW] Missing bonus-disc fixture → FIXED.** `test/album-fixtures.test.ts`
  models a two-disc *deluxe* release with a bonus disc as real album-track
  objects, asserting stable recording identity, distinct track identity across
  the medium boundary, disc numbers, and a preserved free-text (`A4`) position.

Verification after fixes: **46 vitest tests pass**; `pnpm typecheck` and
`pnpm build` clean; built-package runtime probes confirm the rejections above.
