# Reference Addons and SDK Re-audit

- **Audit ID:** A-006
- **Status:** RECONCILED — all 6 findings addressed 2026-07-20 (see Resolution); re-audit invited to confirm
- **Supersedes:** A-005 for current implementation sign-off; partially confirms A-005 reconciliation
- **Audited commits:** `.github` `33db8ae7e22739c060bd1786337b2353191443fe`; `addon-sdk` `ceb1d8b3715811d36b09a051cc4ef4ccc7394f20`; `addons` `afa065fc427d7159d560c8816aa6dd28e1a69b1c`; `player` `af63ee23746ff6d601aa420439bed5540845c6dd`; `backend` `682adc7ed6b5db10d37db9b8a344b65b663e17f9`
- **Last updated:** 2026-07-19

## Scope

This pass re-audits the five A-005 SDK fixes and audits the first Phase 3
reference addons, `stream-legal` and `musicmeta`, including their composition
from MusicBrainz discovery through recording identity to a resolved legal
stream. It covers source licensing/allowlisting, protocol identity, external
API behavior, matching, partial failure, caching, and the SDK boundary. Player,
remaining addons, and backend are still scaffolding. Verification included
source/package inspection, adversarial built-package probes, 111 tests across
the SDK/protocol/addons workspaces, typecheck, build, upstream primary-source
documentation, and all six contract lenses.

## Overall verdict

The implementation does not pass. The A-005 secret-cache fix is incomplete:
configured non-GET requests return a heuristically cacheable 405 before the
router detects the secret-bearing path. `stream-legal` does satisfy the literal
closed-source-set invariant by using only the specifically approved Internet
Archive/Jamendo catalogs, but its item-level “Legal Streams” promise is stronger
than the evidence it checks: matching Archive audio is accepted even when no
license is present. The recording/track
identity model, HTTPS enforcement, configured GET no-store behavior, opaque
client errors, and malformed-input handling are sound. Four medium defects
remain at composition and operations boundaries: wrong-artist tracks pass the
matcher, upstream outages become long-lived cached empty results, MusicBrainz's
mandatory one-request-per-second limit is not enforced, and meta route type/ID
contradictions are accepted and publicly cached.

## Findings

### [MEDIUM] “Legal Streams” overstates item-level license certainty

- **Category:** legal-validity
- **Repo / file / line:** `addons/packages/stream-legal/src/sources/internet-archive.ts:34-73`; `addons/packages/stream-legal/src/match.ts:65-78`
- **Reference:** no existing invariant — new. Checklist §5's literal catalog-level allowlist is satisfied; this finding concerns the stronger README/manifest item-level claim.
- **Finding:** The Internet Archive query filters only by title, creator, and `mediatype:audio`. `filesFor` copies `metadata.licenseurl` when present but never requires it or validates it against an approved CC/public-domain set. `rankCandidates` checks only HTTPS and match score. A built-package probe supplied a matching item/file with no license metadata and the addon emitted `https://archive.org/download/unlicensed/song.mp3` as a candidate. This does not violate the checklist's explicit approval of Internet Archive as a catalog, but it does not substantiate the addon's stronger “Direct streams from Creative-Commons & public-domain catalogs” / “Legal Streams” presentation at item level.
- **Why it matters:** Source-level provenance is not the same as item-level license certainty. A user may reasonably read the name and description as confirmation that each returned recording has verified open rights, when the implementation only confirms that it came from an approved host.
- **Suggested fix:** Fail closed per item/file. Require an explicit recognized CC/public-domain rights value from audited metadata, normalize and allowlist accepted license identifiers/URLs, and add rejection tests for absent, unknown, malformed, and all-rights-reserved metadata. Do not infer permission from Archive hosting alone.
- **Verdict:** CONFIRMED

### [CRITICAL] Configured 405 responses bypass the secret no-store policy

- **Category:** technical-soundness
- **Repo / file / line:** `addon-sdk/packages/sdk/src/router.ts:97-108`
- **Reference:** Checklist §7 configured-addon URLs must be excluded from HTTP/SW caching; A-005 critical finding #1
- **Finding:** Method handling occurs before parsing the request path and computing `hasConfigSegment`. A `POST` to `/<encoded-secret>/stream/...` therefore returns 405 with no `Cache-Control`, rather than `no-store, private`; a runtime probe confirmed it. This contradicts the new source comment's promise that every secret-bearing response uses no-store. RFC 9110 §15.5.6 defines 405 responses as heuristically cacheable unless explicit controls say otherwise.
- **Why it matters:** The configured request target itself contains the debrid key. A client, proxy, or CDN can retain the 405 cache entry and its secret-bearing cache key even though successful configured GET responses were hardened. An attacker or accidental client method is enough to re-open the credential-retention path A-005 intended to close.
- **Suggested fix:** Parse/detect the secret-bearing path before every early return, including OPTIONS and method rejection, and merge `no-store, private` into every response for that path. Add method-matrix tests over configured manifest/configure/resource/error routes.
- **Verdict:** CONFIRMED

### [MEDIUM] Exact-title results from the wrong artist clear the match threshold

- **Category:** end-to-end-user-value
- **Repo / file / line:** `addons/packages/stream-legal/src/match.ts:34-62`
- **Reference:** no existing invariant — new; the addon's documented promise that weak/unrelated matches are dropped
- **Finding:** The weighted sum lets title dominate without any minimum artist agreement. An exact title (0.6) plus matching duration (0.1) already exceeds the 0.5 threshold even when artist similarity is zero. A built-package probe for `Correct Artist — Home` scored a same-duration `Wrong Artist — Home` at 0.8 and would stream it.
- **Why it matters:** Common song titles such as “Home”, “Intro”, or “Stay” can resolve to a different artist's recording. The user presses play on one MusicBrainz recording and hears a confidently labeled but unrelated song—the core journey's worst non-error outcome because it looks successful.
- **Suggested fix:** Require a minimum artist score when artist metadata is available, then apply the combined rank; use duration only as corroboration. Add same-title/different-artist, cover/version, punctuation, alias, and missing-artist fixtures.
- **Verdict:** CONFIRMED

### [MEDIUM] Complete source outages become six-hour cached empty results

- **Category:** system-design
- **Repo / file / line:** `addons/packages/stream-legal/src/resolve.ts:32-38`; `addons/packages/stream-legal/src/handler.ts:12-15`
- **Reference:** Contract §2a outage/partial-results journey; Protocol §6 genuine failure vs no results
- **Finding:** `Promise.allSettled` discards every rejected source without retaining success/failure state. If all sources fail, resolution returns `[]`, and the handler labels it a successful response with `Cache-Control: public, max-age=21600, stale-while-revalidate=86400`. A legitimate no-match and a total upstream outage are indistinguishable.
- **Why it matters:** A transient Internet Archive/Jamendo outage can poison shared/browser caches with “no streams” for six hours (and stale reuse for a day). Users see a silent dead end long after the upstream recovers, while monitoring sees a successful 200 instead of an actionable dependency failure.
- **Suggested fix:** Preserve source outcomes. If every attempted source fails, throw/return a retryable 5xx with no-store or short retry metadata; for partial success, return results with a conservative cache policy. Cache genuine empty searches separately and briefly.
- **Verdict:** CONFIRMED

### [MEDIUM] MusicBrainz clients do not enforce the upstream rate limit

- **Category:** system-design
- **Repo / file / line:** `addons/packages/musicmeta/src/musicbrainz.ts:49-123`; `addons/packages/stream-legal/src/metadata.ts:21-49`
- **Reference:** no existing invariant — new; MusicBrainz API operational contract
- **Finding:** Both addons call MusicBrainz directly for every request with no shared limiter, queue, retry/backoff, or request coalescing. The official API requires applications to stay at no more than one call per second per source IP and responds with 503/blocking when exceeded. Two independently served addons behind one host make the violation easier, and normal concurrent search/play activity is enough to exceed it. See [MusicBrainz API](https://musicbrainz.org/doc/MusicBrainz_API) and [rate-limiting rules](https://musicbrainz.org/doc/MusicBrainz_API/Rate_Limiting).
- **Why it matters:** The first discovery-to-play journey requires MusicBrainz twice: metadata search/detail and stream metadata lookup. Concurrent users or even quick interactions trigger throttling, causing catalog 500s and stream failures at the primary external boundary. The implementation is not operationally viable as a hosted addon.
- **Suggested fix:** Put MusicBrainz access behind a process/deployment-wide one-request-per-second scheduler with deduplication, bounded queueing, caching, and 503 `Retry-After` backoff. Share the gateway/limiter across both addons when co-hosted, or use a compliant mirror/database for scale. Test concurrency deterministically.
- **Verdict:** CONFIRMED

### [MEDIUM] Meta route type and ID can contradict each other

- **Category:** technical-soundness
- **Repo / file / line:** `addon-sdk/packages/sdk/src/router.ts:168-172`; `addons/packages/musicmeta/src/handler.ts:23-28`; `addons/packages/musicmeta/src/meta.ts:25-60`
- **Reference:** Checklist §6 content type ↔ ID identity; Protocol §3/§5
- **Finding:** The SDK validates that the meta route type is a known enum but does not validate its pairing with the ID. `musicmeta` then intentionally ignores the route type and dispatches only on ID entity. A runtime probe of `/meta/artist/mbid:recording:<uuid>.json` returned 200 with a `type:"track"` response and `Cache-Control: public, max-age=86400`.
- **Why it matters:** The wire URL says one entity while the response says another. Clients and intermediaries can key routing/cache state by the request type and persist a contradictory object for a day, recreating the identity ambiguity the discriminated response schema was added to prevent.
- **Suggested fix:** Add resource-specific request validation: meta must enforce artist→artist MBID, album→release MBID, track→recording MBID/ISRC, playlist→playlist ID before handler invocation. Make handler arguments discriminated accordingly and add every mismatch permutation as a route test.
- **Verdict:** CONFIRMED

## Six-lens summary

- **Technical soundness:** Changes required: the SDK secret-cache boundary is incomplete and meta request identity is contradictory. A-005's configured GET caching, opaque error, type enum, configuration-required, and malformed-encoding fixes are otherwise confirmed.
- **Legal validity:** The literal implemented invariants pass: sources are the fixed approved Internet Archive/Jamendo set, no audio bytes are stored, and no arbitrary request URL is proxied. One new medium product-trust finding flags that item-level “Legal Streams” language is stronger than the rights evidence checked. Unimplemented debrid/YouTube invariants were not failed.
- **Overall implementation quality:** All tests/typechecks/builds pass and dependency injection keeps source adapters testable. Boundary tests omit unlicensed records, method-matrix caching, same-title wrong-artist matches, all-source failure semantics, concurrency limits, and route type/ID contradictions.
- **System design and operations:** Changes required for outage semantics and MusicBrainz rate compliance. There are also no explicit upstream timeouts, but that is not separately counted because the implemented handler API does not yet propagate request cancellation and the confirmed rate/outage findings already block this slice.
- **UI/UX and accessibility:** No product UI has landed. Addon responses are machine-facing; no separate accessibility finding applies. Wrong-track success and cached outage empties would produce dishonest player states once UI integration lands.
- **End-to-end user value and delight:** The happy-path `musicmeta` recording ID → `stream-legal` HTTPS URL composition works, but legal trust, matching correctness, and recovery are not reliable enough for the primary press-play journey.

## Verification record

- `addon-sdk`: 78 tests passed (46 protocol + 32 SDK), typecheck and build passed; localhost permission was required only for the intended live HTTP test.
- `addons`: 33 tests passed (17 musicmeta + 16 stream-legal), typecheck and build passed.
- Built-package probes confirmed: an unlicensed IA item is emitted; configured POST returns 405 without cache control; exact-title/wrong-artist score is 0.8; contradictory meta route returns a track response under an artist URL with a one-day public cache.
- Source inspection confirmed all-source rejection collapses to a cacheable empty 200 and both MusicBrainz clients contain no limiter/retry/backoff.
- No code in `player`, `addon-sdk`, `addons`, or `backend` was modified.
- No GitHub issues were created; publication awaits explicit user approval.

## Resolution (2026-07-20, implementer)

All six findings addressed; docs synced (PROTOCOL.md §6/§7, Review Checklist
§5/§6/§6a + status, both `CLAUDE.md`s, plan Phase 3, memory).

- **[CRITICAL] Configured 405 bypassed no-store → FIXED.** `router.ts` computes
  the secret-bearing path (`hasConfigSegment` → `cachePolicy`) **before** the
  method/OPTIONS early-returns, and merges `no-store, private` into the 204 and
  405 responses. New method-matrix tests (`POST/PUT/DELETE`/`OPTIONS` over
  configured vs unconfigured paths).
- **[MEDIUM] "Legal Streams" overstated item license → FIXED.** New
  `stream-legal/src/license.ts` (`isRecognizedOpenLicense`, fail closed). The
  Internet Archive and Jamendo sources emit a candidate only when the item
  carries a recognized CC/public-domain value; `rankCandidates` re-checks
  centrally. Tests: absent/unknown/malformed/all-rights-reserved rejected;
  CC + public-domain accepted; unlicensed IA item → `[]`. The stream name now
  shows the license (e.g. `CC BY 3.0`).
- **[MEDIUM] Wrong-artist same-title matches → FIXED.** `match.ts` requires a
  minimum artist score (`MIN_ARTIST_SCORE`) when artist metadata exists on both
  sides; duration is corroboration only. Probe `Correct Artist — Home` vs
  `Wrong Artist — Home` now drops the wrong artist.
- **[MEDIUM] Total outage → 6h cached empty → FIXED.** `resolveStreams` returns
  `{ streams, allSourcesFailed }`; the handler throws on a total outage
  (uncacheable 500, not heuristically cached) and caches a genuine no-match only
  briefly (`max-age=300`). Metadata-lookup failures propagate as errors too.
- **[MEDIUM] MusicBrainz rate limit unenforced → FIXED.** New shared
  **`@p2p-songs/musicbrainz`** package: a `RateLimiter` (≤1 req/sec, injectable
  clock) + `503 Retry-After` backoff, wrapping every MB request. Both
  `musicmeta` and `stream-legal` now consume it (dedup of the former per-addon
  clients); co-hosted addons can share one limiter instance. Documented caveat:
  separate processes each hold their own per-IP budget — use an external gateway
  or MB mirror for true multi-process scale.
- **[MEDIUM] Meta route type/id contradiction → FIXED.** `router.ts` validates
  the `meta` route type against the id entity (`metaIdMatchesType`) before
  invoking the handler; a mismatch is a 404. Permutation tests added.

Verification: **SDK 36 tests; addons 46 tests (musicbrainz 7 + musicmeta 14 +
stream-legal 25)**; typecheck + build clean; built-package probes replay the
audit's runtime cases — configured POST → `no-store, private`;
`/meta/artist/<recordingId>` → 404; unlicensed candidate dropped; wrong-artist
dropped.
