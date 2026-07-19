# Review Checklist — Invariants for Adversarial Review

This project is built under a two-role split: one agent/session implements,
a separate agent audits without access to the implementer's conversation
history — only what's committed to these repos. This doc exists so that
audit has a precise, mechanically-checkable list to work from, instead of
having to re-derive intent from [`IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md)
prose each time. When the two disagree, `IMPLEMENTATION_PLAN.md` is the
source of truth for *why*; this doc is the source of truth for *what to check*.
For the full audit process (roles, ground rules, finding format, how
findings get delivered back) see
[`ADVERSARIAL_REVIEW_CONTRACT.md`](./ADVERSARIAL_REVIEW_CONTRACT.md).

Each item names which repo(s) it applies to and which plan section it comes from.

## 1. Protocol neutrality (`player`, `addon-sdk`)
- [ ] The `player` (its `src/core` engine) never bundles, default-installs, or
      hardcodes *any* specific stream addon (including `stream-debrid`). Addon
      installation is exclusively "user pastes a manifest URL." — Plan §3
- [ ] `addon-sdk` imposes no assumptions about what kind of stream source
      an addon built with it uses — it's transport/protocol tooling only,
      content-agnostic. — Plan §1, §3

## 2. `stream-debrid` shape (`addons`)
- [ ] `stream-debrid` is **one self-contained addon**: its own discovery
      logic, its own aggregation/ranking, its own debrid resolution, its
      own `/configure` page. — Plan §2
- [ ] It does **not** implement a generic provider/plugin interface for
      fanning requests out to *other, separately-hosted* addons and
      merging their results (the AIOStreams shape). That pattern was
      explicitly tried and reverted — if it reappears, it's a regression,
      not a feature. — Plan §2
- [ ] **Within-album-torrent file selection is explicit, not "largest file."**
      Requests are keyed by `mbid:recording:` but music torrents are whole
      albums; `stream-debrid` selects the correct track file by disc+track
      position when album context (`mbid:track:`/`mbid:release:`) is present,
      else by title+duration match. The player still receives only a resolved
      `url` (file selection is internal). — Plan §2a, §8

## 3. `stream-debrid` legal invariants (`addons`)
- [ ] It never persists or caches resolved **audio bytes** on its own
      infrastructure. It may hold candidate *metadata* (title, infoHash,
      file size) but never the file content itself. — Plan §3
- [ ] Every debrid API call (`checkCache`, `resolve`, etc.) uses the
      debrid credentials supplied in **that request's own** `/configure`
      config — never an operator-owned, shared, or pooled account paid for
      by whoever deploys the addon. — Plan §3
- [ ] It doesn't choose or ship a hardcoded indexer/tracker source that
      can't be reasoned about — built-in discovery logic is fine (Torrentio
      does this too), but nothing that looks like laundering a specific
      illicit source as "the product." — Plan §3
- [ ] Public hosting of the *code/service* is fine (Torrentio itself is
      publicly hosted) as long as the two invariants above hold — don't
      flag public deployment by itself as the problem. — Plan §3

## 4. `stream-ytmusic` (`addons`)
- [ ] Returns Stremio's native `ytId`-style pointer (official YouTube
      IFrame/Music embed playback), **not** a raw extracted audio stream
      URL via `yt-dlp`/similar by default. Raw extraction, if ever added,
      must be an explicitly-labeled non-default option, not the addon's
      normal behavior. — Plan §4

## 5. `stream-legal` (`addons`)
- [ ] Only pulls from confirmed CC-licensed/open catalogs (Jamendo,
      Internet Archive, Free Music Archive). — Plan §4, §7
- [ ] Does not proxy or resolve arbitrary user-supplied URLs — it's a
      fixed, audited set of sources, not an open redirector. — Plan §7

## 6. Protocol/ID conformance (`addon-sdk`, `addons`)
- [ ] A **standalone, versioned** wire spec exists at `addon-sdk/docs/PROTOCOL.md`
      (routes, payloads, error behavior, examples), and it agrees with the
      `@p2p-songs/protocol` schemas. — Plan §8, §10 (Phase 1)
- [ ] Content types are `artist`/`album`/`track`/`playlist`; resources are
      `catalog`/`meta`/`stream`/`lyrics`. — Plan §8
- [ ] IDs are **entity-typed** — `mbid:<entity>:<uuid>` with entity ∈
      `artist`/`release`/`recording`/`track`; `isrc:` secondary; `playlist:<token>`
      (addon-scoped opaque, no colon) for playlists. The old synthetic
      `mbid:<release-mbid>:<track-number>` form is **removed** (it collides
      across discs and breaks on free-text vinyl numbers) — flag it if it
      reappears. — Plan §8
- [ ] **Content type ↔ id identity is enforced**: `meta` is a discriminated
      union where `artist`→artist MBID, `album`→release MBID, `track`→recording
      MBID/ISRC, `playlist`→playlist id. A type carrying a foreign id is rejected
      on the wire. — Plan §8 (audit A-004)
- [ ] **Every resource URL is `https://`** — `stream.url`, `lyric.url`, `poster`,
      `logo`, `background` reject `http`/`ftp`/`file`/`data`/`javascript`;
      `ytId`/`infoHash` exempt (not URLs). — Plan §8 (audit A-004)
- [ ] **`stream`/`lyrics` are keyed by `mbid:recording:<uuid>`** (the streamable
      unit), not by a release-track. `mbid:track:<uuid>` may accompany a request
      as album context only. Never resolve a stream against a bare
      release+track-number. — Plan §8
- [ ] SDK protocol schema tests cover the identity fixtures: **multi-disc**
      (disc1-t1 ≠ disc2-t1), **vinyl/free-text number** (`A4`), **bonus disc**,
      and **same recording on two releases** (one recording id, two track ids).
      — Plan §8
- [ ] Stream objects from `stream-legal`/`stream-debrid` are always fully
      resolved (`url` present); a bare `infoHash`/`fileIdx` pointer with no
      `url` should not come from any reference addon. — Plan §8
- [ ] The optional link-expiry hint (`behaviorHints.expiresAt` UTC ISO-8601 /
      `maxAgeSeconds` int) is validated by the SDK when present, and is treated
      everywhere as an **optional hint** — never a required field, never the
      basis of a correctness guarantee. — Plan §8; ARCHITECTURE §5a
- [ ] **The SDK router validates the route content `type`**: `stream`/`lyrics`
      require literal `track`; `catalog`/`meta` require a protocol `ContentType`;
      anything else is a 404, never a handler call with a contradictory type.
      Malformed percent-encoding / bad requests are controlled `4xx`, never an
      escaped `URIError`/500. — Protocol §5/§6 (audit A-005)

### 6a. SDK credential-boundary safety (`addon-sdk`) — audit A-005
- [ ] A request whose path carries a config segment is **secret-bearing**: its
      manifest, `/configure`, and resource responses are `Cache-Control:
      no-store, private` — **never** `public`/shared caching (the segment holds
      the debrid key). Unconfigured requests may cache normally. — Checklist §7
- [ ] **Error bodies are opaque** — a failure returns a stable `err` string
      only, never a handler/provider exception message (which can contain the
      credential). Diagnostics go only to the opt-in `onError` hook. — Checklist §7
- [ ] **`configurationRequired` fails closed**: the router rejects a resource
      request (400) unless a valid config decoded — a handler is never invoked
      without credentials, so no fallback to an operator account is possible. A
      malformed config prefix is a 400, not a silent downgrade. — Plan §3; §3 above

## 7. Secrets hygiene (all repos)
- [ ] No debrid API keys, indexer credentials, or other secrets are ever
      committed to any repo, used as a default/fallback, or logged. Config
      lives only in the per-request `/configure`-encoded value.
- [ ] **Player-side reality (corrected):** a *configured* stream addon's
      manifest URL contains the user's debrid key, and the player necessarily
      holds it to call the addon — so the player must treat configured addon
      URLs as **secrets**: stored in a secret-bearing store, never
      logged/exported/telemetered, config segment redacted in any UI display,
      and excluded from service-worker/HTTP caching. Do **not** accept (or
      write) a claim that "the player never holds the key" — that was audited
      as false. — Plan §7; ARCHITECTURE §6a
- [ ] **Sync of the secret is now permitted, under conditions (updated
      2026-07-17):** the earlier "configured URLs are never synced" rule is
      **intentionally reversed** — logged in, they sync to the user's own
      backend (server-readable, Stremio's model). This is NOT a regression to
      flag *provided* it stays: (a) optional (app works logged-out), (b)
      self-hosted as the supported model, (c) encryption-at-rest + TLS + RLS,
      (d) the client-side rules above still hold. — Plan §3/§7; ARCHITECTURE §6b
- [ ] The secret store is **not** called a "keychain" and is not presented as a
      security boundary (same-origin script can read it). A v1 browser threat
      model is required: strict CSP (no `unsafe-inline`/`eval`), Trusted Types
      where available, redacted error boundaries, minimal in-origin deps, and a
      test asserting configured URLs never land in any SW/HTTP cache. Client-
      side encryption without a user-held key is **not** accepted as a fix. —
      ARCHITECTURE §6a
- [ ] **No remote UI code (`player`):** themes/plugins are first-party bundled,
      build-time code only — never fetched/`eval`'d at runtime. Remote theme
      code would run in the origin holding the debrid credential. External/
      distributable themes are a v1 non-goal. — ARCHITECTURE §6a, §7a

## 8. Core engine (`player`)
The player architecture is specified in detail in
[`player/docs/ARCHITECTURE.md`](https://github.com/p2p-songs/player/blob/main/docs/ARCHITECTURE.md),
which **deliberately supersedes** the master plan's "Elm-style core" idea.
Audit the player against that doc, not against a stremio-core port.
- [ ] The core engine (`src/core`) imports nothing from `src/ui` — the
      headless/UI-agnostic boundary is real and lint-enforced. Engine logic
      (queue, playback machine, scheduler) must be unit-testable without a
      browser or a live addon. — ARCHITECTURE §8
- [ ] **Do NOT flag the absence of an Elm `Msg→Effects→Model` runtime as
      drift.** It was intentionally retired (ARCHITECTURE §1, §11);
      predictable state is provided by the scoped playback state machine
      instead. A Rust/WASM port remains out of scope for web-only v1.
- [ ] Stream resolution is just-in-time (next 1-2 items only), never
      whole-queue-upfront. Resolving the entire queue eagerly (expiring
      debrid links, hammering debrid APIs) is an anti-pattern — flag it.
      — ARCHITECTURE §5, §11
- [ ] Resolved media is memory-only: resolved stream URLs /
      `QueueItem.resolution` and the `/stream` query cache are never persisted;
      queue items hydrate to `resolution: idle` and re-resolve. Persisting
      bearer stream links is a defect. — ARCHITECTURE §6, §11
- [ ] Configured addon URLs are handled as secrets in the player (see §7
      above) — not the false "no keys in the player" claim. — ARCHITECTURE §6a
- [ ] "Gapless" is validated as a measured target (silence threshold on a
      browser×codec matrix, same-origin test fixtures), not asserted as an
      absolute; crossfade is the documented fallback. — ARCHITECTURE §4c
- [ ] **`/stream` is a command, not a cached query:** it does NOT run under the
      generic TanStack Query policy (auto-retry, `refetchOnWindowFocus`,
      `refetchOnReconnect`, SWR). It uses a scheduler-owned command plane with
      in-flight dedup, `retry: false`, memory-only results. Metadata calls
      (manifest/catalog/meta/lyrics) and `/stream` must be separate planes. —
      ARCHITECTURE §5a, §6
- [ ] **Stream freshness is re-resolve-on-failure**, not dependence on a
      protocol expiry field (which is only an optional hint). — ARCHITECTURE §5/§5a
- [ ] **Async results commit by identity, not just abort:** every resolve/load
      carries `{sessionEpoch, queueItemId, attemptId}` and the reducer drops
      completions whose stamp doesn't match current state. AbortController alone
      is insufficient. Race tests required (resolve-after-skip,
      failure-after-success, reorder-during-resolve, double-completion). —
      ARCHITECTURE §4b
- [ ] **Queue identity is by stable ID:** `currentItemId` + `playOrder` (not a
      mutable array index); "up next" reads from `playOrder` so it's correct
      under shuffle. Index-as-identity is a defect. — ARCHITECTURE §4a
- [ ] **Failure is bounded:** skip-ahead runs inside a per-session failure
      sweep with a terminal error state and provider backoff; `repeat: "all"` /
      autoplay must not create an unbounded resolve/fail/skip loop. —
      ARCHITECTURE §4b

## 9. Accounts & sync backend (`backend`, `player`)
Applies once Phase 5b lands; the `backend` repo is optional and self-hosted.
- [ ] **Login is optional / app is local-first:** the player must work fully
      without an account; sync is additive, never a gate. — ARCHITECTURE §6b
- [ ] **Per-user isolation via RLS:** Postgres Row-Level Security ensures a
      user can only ever read/write their own rows. No endpoint returns another
      user's data. — ARCHITECTURE §6b
- [ ] **Service key never ships to the client;** only the anon/public key does.
      — ARCHITECTURE §6b
- [ ] **Secret at rest:** synced configured-URL/config columns are encrypted at
      rest; TLS in transit. Server-readable is accepted by decision, but
      plaintext-at-rest in the DB file is not. — ARCHITECTURE §6b; Plan §7
- [ ] **Resolved media never syncs:** resolved stream URLs / `QueueItem.resolution`
      / `/stream` results are never sent to the backend — only queue *identity*
      and durable library/settings. — ARCHITECTURE §6/§6b
- [ ] **Self-hostable, no proprietary lock-in:** the backend must deploy on a
      rented server (Docker); no Firebase or non-self-hostable dependency. A
      public multi-tenant deployment is an un-blessed, different-posture choice
      (operator holds many users' keys) — Plan §3. — ARCHITECTURE §6b

## Current status (update as phases land)
**Start here:** the newest-first
[`docs/audits/README.md`](./audits/README.md) registry is authoritative for the
latest audit, supersession, sign-off, and open findings. The prose below is the
chronological history.

**Implementation started (2026-07-18).** First code landed:
**`@p2p-songs/protocol`** in the `addon-sdk` repo (`packages/protocol`) — the
schema-first (zod) wire contract: entity-typed MBIDs, stream object + optional
expiry hint, recording-keyed stream/lyrics requests, catalog/meta/manifest
schemas. 26 vitest tests pass, including the A-003 identity fixtures
(multi-disc, vinyl free-text, same-recording-on-two-releases); typecheck +
build + Node runtime smoke all green. This is the foundation the SDK, addons,
and the player addon-client all import. Everything else is still
planning/scaffolding.

Earlier: the repos (`​.github`, `player`, `addon-sdk`, `addons`, `backend`)
were planning-only. The core-player plan was audited on 2026-07-17 (verdict:
changes
required — 2 high, 2 medium); **all four findings have since been reconciled
into the docs** (credential model corrected, master plan de-Elm'd and moved to
the real 4-repo layout, resolved-media persistence made memory-only, gapless
reframed as a measured target). See the Resolution section of
[`docs/audits/2026-07-17-core-player-plan.md`](./audits/2026-07-17-core-player-plan.md).
Those original findings are closed.

**Architecture re-audit (2026-07-17): changes required — 2 high, 4 medium.**
The revised macro-architecture was judged sound; the six findings concerned
implementation-shaping seams. **All six are now reconciled into the docs**
(2026-07-17): optional protocol expiry hint added (Plan §8) with
re-resolve-on-failure as the real guarantee; `/stream` split into a
scheduler-owned command plane distinct from the metadata query plane
(ARCHITECTURE §5a/§6); queue re-modeled on stable IDs + `playOrder`
(§4a); async completions commit by `{sessionEpoch, queueItemId, attemptId}`
stamp (§4b); credential store renamed secret-bearing with a v1 browser threat
model + no-remote-theme-code invariant (§6a/§7a); failure skip-ahead bounded by
a per-session sweep + backoff (§4b). New invariants added to §6/§7/§8 above.
See the Resolution section of
[`docs/audits/2026-07-17-core-player-architecture-reaudit.md`](./audits/2026-07-17-core-player-architecture-reaudit.md).
Both re-audit issues (player#2, .github#2) closed. **At the close of A-002**
there were no blocking findings; A-003 below is newer and owns current sign-off.

**Design change (2026-07-17): optional accounts & sync added.** The original
"no server, local-only" assumption was reversed by product decision: users can
log in to sync addons + listening state across devices via an **optional,
self-hosted** backend (self-hosted Supabase). The credential-sync model is
**server-readable, not zero-knowledge** (Stremio's model), made responsible by
self-hosting + encryption-at-rest + TLS + RLS. New `backend` repo; new
checklist §9; the "never synced" invariant is intentionally reversed (see §7).
See ARCHITECTURE §6b and Plan §3/§7/§9/§10 (Phase 5b). This design is included
in the product-wide plan audit below; re-audit again when Phase 5b code lands.

**Product-wide plan audit (2026-07-17, revised after scope clarification): does
not pass — 1 high.** The technical macro-architecture and documented legal
invariants are sound. Manifest-only addon installation is intentional;
accessibility, advanced offline conflict handling, advanced encryption key
lifecycle, and concurrent-device playback are deferred and are not findings.
Basic encryption at rest remains required. The sole blocker is the album-track
ID scheme, which collides across MusicBrainz media and does not clearly separate
track identity from recording identity. See
[`docs/audits/2026-07-17-product-wide-plan.md`](./audits/2026-07-17-product-wide-plan.md).

**A-003 resolved (2026-07-18).** The album-track ID finding is reconciled:
IDs are now **entity-typed** (`mbid:artist|release|recording|track:<uuid>`),
the synthetic `release:track-number` composite is removed, **recording is the
streamable/cache/dedup unit** (`stream`/`lyrics` keyed by `mbid:recording:`),
`mbid:track:` is album context only, and multi-disc/vinyl/same-recording
fixtures are required (§6 above; Plan §5/§8; addon-sdk contract; ARCHITECTURE
§4a). addon-sdk#1 closed. **Plan is signed off for implementation under A-003's
declared scope.** Re-audit when the first vertical slice of code lands.

**First implementation audit A-004 (2026-07-19): changes required — 3 medium,
1 low.** The `@p2p-songs/protocol` package passes its tests, typecheck, and
build, and correctly represents the central recording/track identity split.
Current implementation sign-off was blocked because direct resource schemas
accepted arbitrary URL schemes, metadata permitted contradictory content-type/ID
pairs while playlist identity had no valid namespace, the Phase 1 standalone
wire specification was absent, and the required bonus-disc fixture was not
tested. See [`docs/audits/2026-07-19-protocol-implementation.md`](./audits/2026-07-19-protocol-implementation.md).

**A-004 reconciled (2026-07-19).** All four findings addressed in
`@p2p-songs/protocol`: (1) a shared `httpsUrlSchema` restricts `stream.url`,
`lyric.url`, `poster`, `logo`, `background` to `https://` (rejects
http/ftp/file/data/javascript), with negative tests; (2) `meta` is now a
discriminated union enforcing type↔id identity, and `playlist` gets its own
`playlist:<token>` addon-scoped namespace (Plan §8; §6 above); (3) the standalone
versioned wire spec now exists at `addon-sdk/docs/PROTOCOL.md`; (4) an explicit
multi-disc + bonus-disc album-track fixture was added. **46 vitest tests pass;
typecheck + build + built-package runtime probes green.** Re-audit to confirm.

**Phase 2 — `@p2p-songs/addon-sdk` implemented (2026-07-19).** `packages/sdk`:
`AddonBuilder` (manifest validation + handler-registration guards), typed
`define*Handler`s, a **framework-agnostic** `createRouter` (`{method,url}` ⇒
`{status,headers,body}`) with a thin `serveHTTP` node:http adapter over it (no
Express — an intentional divergence from the plan's original wording), and the
`/configure` round-trip (`encodeConfig`/`decodeConfig`, base64url path segment,
default configure page). The router enforces the protocol at the boundary: CORS
+ OPTIONS, recording-keyed stream/lyrics (400 on a non-recording id), and
**response-schema validation** (an invalid handler response is a 500). 22 SDK
tests incl. a live hello-world served over HTTP; 68 total across the workspace;
typecheck + build green.

**A-005 addon SDK implementation audit (2026-07-19): changes required — 2
critical, 3 medium.** A-004 reconciliation is confirmed. Phase 2 was not signed
off: configured secret-bearing paths were marked public-cacheable; raw handler
exception messages could disclose credentials; stream/lyrics route types were not
enforced; `configurationRequired` failed open; and malformed percent encoding
escaped the router response boundary. See
[`docs/audits/2026-07-19-addon-sdk-implementation.md`](./audits/2026-07-19-addon-sdk-implementation.md).

**A-005 reconciled (2026-07-19).** All five addressed in the SDK router/serve:
secret-bearing configured paths → `no-store, private` (never public); client
error bodies opaque (`{ err }`) with an opt-in `onError` diagnostics hook so
exception messages never reach callers; route content types validated
(stream/lyrics require `track`); `configurationRequired` fails closed (400) with
a malformed config prefix → 400; malformed percent-encoding → controlled 400.
New invariants in §6/§6a above. **32 SDK tests (10 new A-005 regressions in
`test/security.test.ts`); 78 total; typecheck + build + built-package
adversarial probes green.** Re-audit to confirm.

**A-006 reference addons + SDK re-audit (2026-07-19): changes required — 1
critical, 5 medium.** The SDK secret-cache fix misses method rejection:
configured 405 responses lack `no-store`. `stream-legal` satisfies the literal
fixed-catalog invariant, but its item-level “Legal Streams” language exceeds
the rights evidence it checks (new product-trust finding). Additional findings:
same-title wrong-artist matches pass; complete
source outages become six-hour cached empty results; MusicBrainz's required
rate limit is unenforced; and meta route type/ID contradictions are accepted.
See [`docs/audits/2026-07-19-reference-addons.md`](./audits/2026-07-19-reference-addons.md).
