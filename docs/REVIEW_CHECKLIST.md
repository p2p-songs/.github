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
> **Naming:** `stream-debrid` is the **role**; the implementation is
> **`bitbop`** (`addons/packages/bitbop`, `@p2p-songs/bitbop`, manifest id
> `com.p2p-songs.bitbop`). Every `stream-debrid` item in §2 and §3 is checked
> against Bitbop. Implemented 2026-07-21.

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
- [ ] **The caller-supplied indexer URL is policed before it is fetched
      (audit A-011).** "The indexer is the user's own" means the addon fetches a
      URL an arbitrary caller controls, so a publicly-hosted instance is an SSRF
      proxy unless the destination is checked. Required, and all four parts are
      load-bearing:
      - **Scheme policy** — https only in public mode.
      - **Every redirect hop is re-validated.** A permitted public URL that
        302s to `http://127.0.0.1` defeats a pre-flight-only check, and `fetch`
        follows redirects by default. Manual redirect handling + a hop cap.
      - **The validated address must be the connected address.** Validating DNS
        and then letting the HTTP stack resolve again leaves a rebinding window.
        Bitbop uses `node:http`'s `lookup` hook so there is exactly one
        resolution; a hostname with any non-public address in its answer is
        refused wholesale.
      - **Literal IP hosts are validated separately.** Node skips DNS for a
        numeric host, so a `lookup`-hook-only guard misses
        `https://169.254.169.254/…` entirely — including bracketed IPv6 and
        IPv4-mapped `::ffff:127.0.0.1`. This one bit the first implementation;
        check it explicitly.
      - **Addresses are classified by their bits, not their text (audit
        A-012).** One address has many spellings and the attacker chooses;
        `::ffff:7f00:1`, `0:0:0:0:0:ffff:7f00:1` and `::ffff:127.0.0.1` are the
        same loopback. Prefix/regex matching on the string is the bug — and note
        `new URL()` rewrites the dotted form to hex, so a check that only
        recognizes `::ffff:127.0.0.1` matches the one spelling that can never
        arrive. Require a numeric IPv6 parse that judges mapped/compatible/
        translated embedded v4 in every notation, with tests for the hex forms.
      **The policy is about _who chose the destination_, not about whether the
      destination is private.** A caller-supplied URL fetched server-side is a
      confused deputy and must be policed; an **operator-supplied** upstream
      (the operator's own Prowlarr on their own internal network) is an ordinary
      configured dependency and a private address there is correct. If an
      operator-configured indexer mode exists, check that a caller-supplied URL
      is **refused** rather than merged or overridden when one is set, and that
      the bypass covers only the operator's URL. See `docs/DEPLOYMENT.md`.
      **Deployment modes:** public-safe must be the **default** (a public
      instance must not be one forgotten env var away from being an open proxy);
      a self-hosted indexer on loopback/LAN is an explicit opt-in
      (`BITBOP_ALLOW_PRIVATE_INDEXERS=1`). Do **not** flag the existence of the
      permissive mode as the finding — flag it only if it is the default, or if
      the active policy isn't stated to the operator and on `/configure`.
      — audit A-011; `addons/packages/bitbop/src/net/`
- [ ] **A cache check never leaves work running on the user's account.**
      Real-Debrid withdrew `/torrents/instantAvailability`, and the state machine
      that replaced it only reveals cachedness *after* file selection — which is
      also what starts a download. So "is this cached?" is unavoidably a write,
      and the invariant is about cleaning up after it:
      - Anything the addon **adds** in order to check must be **deleted** before
        it returns, unless it turned out cached. A torrent the *user* already
        had is never deleted — only the addon's own leavings.
      - Selection is **audio files only**, never `files=all`; a miss must not
        cost the user an entire album's download.
      - The provider-side torrent id is **threaded from check to resolve**, so a
        single resolution never adds the same torrent twice.
      - A **non-mutating** bulk pre-check (`GET /torrents`) runs first, so the
        second and later tracks of an album — the common case — add nothing.
      - Probing that requires an add is **rationed separately** from free probes.
        RD allows 250 req/min and the player prefetches; unbounded fan-out is a
        self-inflicted rate limit.
      This is the "cached-only" contract in Plan §2 actually being honoured: an
      addon that *starts* the downloads it promises never to wait for has broken
      it. — `addons/packages/bitbop/src/debrid/realdebrid.ts`
- [ ] **The settle budget bounds the whole check, not just the polling.** It
      starts before `addMagnet`, because the add, the first `info`, and the
      selection call are three round-trips that otherwise sit outside it. Getting
      this wrong twice is instructive: counting poll *attempts* ignored the
      ~260ms each round-trip costs (a "2.5s" budget ran 4.8s), and then bounding
      by wall clock but starting after the add let misses run 848–6844ms against
      a "3s" budget. Only the cleanup `delete` is exempt — it must run whatever
      time is left. Verify with a *miss*, not a hit: a cached torrent answers
      long before any budget matters.
- [ ] **Provider errors returned in a 200 body are parsed.** Real-Debrid reports
      application failures as `{error, error_code}` with an HTTP 200. Unchecked,
      an auth failure or rate limit surfaces later as an unrelated structural
      complaint, and the resolver's outage/no-match distinction silently
      misclassifies it. Auth codes (RD 8–15) must map to the same
      "credential rejected" path as HTTP 401/403.

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
- [ ] **Per-item license, fail closed (audit A-006):** a candidate is emitted
      only if its own metadata carries a *recognized* open (CC / public-domain)
      license — source-level provenance (hosted on the Archive) is **not**
      sufficient. Absent / unknown / malformed / all-rights-reserved → dropped.
      The "Legal Streams" presentation must not exceed the rights actually
      checked. — Plan §3/§7
- [ ] **Matching requires artist agreement (audit A-006):** an exact title with
      a mismatched artist must not resolve — when artist metadata exists on both
      sides, a minimum artist score is required before rank; duration is
      corroboration only, never a substitute. — end-to-end user value
- [ ] **Outage ≠ no-match (audit A-006):** a *total* source outage returns a
      retryable, uncacheable error (not a long-lived cached empty 200); a genuine
      no-match may return empty but only with a *short* cache. — Protocol §6

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
- [ ] **`meta` route type ↔ id identity is validated on input (audit A-006):**
      `artist`↔artist MBID, `album`↔release MBID, `track`↔recording MBID/ISRC,
      `playlist`↔playlist id; a contradictory pair is a 404, never a handler call
      (the input mirror of the meta discriminated response). — Protocol §3/§5
- [ ] **MusicBrainz access is rate-limited (audit A-006):** addons that call
      MusicBrainz go through a shared ≤1 req/sec limiter with `503 Retry-After`
      backoff (the shared `@p2p-songs/musicbrainz` client); co-hosted addons
      should share one limiter instance. — MusicBrainz API operational contract

### 6a. SDK credential-boundary safety (`addon-sdk`) — audit A-005
- [ ] A request whose path carries a config segment is **secret-bearing**: its
      manifest, `/configure`, and resource responses — **and its `OPTIONS`
      preflight and `405` method rejection** (audit A-006) — are `Cache-Control:
      no-store, private`, **never** `public`/shared caching (the segment holds
      the debrid key). The secret detection runs before any early return, so no
      method/route shortcut leaks a cacheable secret-bearing response.
      Unconfigured requests may cache normally. — Checklist §7
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
      **Gate satisfied 2026-07-21, alongside `bitbop`** (this was the hard gate
      A-010 deferred until a credential-bearing addon existed). What landed, and
      what to check it against:
      - `player/src/app/security/csp.ts` builds the policy; a Vite
        `transformIndexHtml` plugin injects it as a `<meta http-equiv>` so it
        applies on any static host. **Production**: `script-src 'self'` — no
        `unsafe-inline`, no `unsafe-eval` — plus `object-src`/`base-uri`/
        `frame-ancestors`/`form-action` `'none'` and
        `require-trusted-types-for 'script'`. **Dev** relaxes script-src for Vite
        HMR only. Verify against `dist/index.html` after `vite build`, not just
        the source.
      - Vite's **modulepreload polyfill is disabled** (`build.modulePreload.
        polyfill: false`) — it emits an inline `<script>` that `script-src 'self'`
        would otherwise have to accommodate. The built HTML must contain **no**
        inline script.
      - **Honest scope, don't flag as a gap:** `connect-src`/`img-src`/
        `media-src` must permit arbitrary `https:` — addons are user-installed
        URLs on hosts that can't be enumerated ahead of time. CSP here is the
        boundary against *injected* code (the actual §6a threat), not against a
        *trusted* addon's own host. `csp.ts` says so in prose.
      - `installTrustedTypesFallback()` registers a `'default'` policy that
        **passes through with a redacted, deduped warning**. This is deliberately
        a *monitored escape hatch*, not a bypass: the app has no DOM-XSS sinks,
        so it should never fire, and it exists so an unexpected dependency sink
        surfaces loudly instead of white-screening. It is documented as
        to-be-tightened-to-throwing after a real-browser pass. Flag it if it is
        ever presented as the *primary* protection — `script-src` is.
      - `ErrorBoundary` renders a generic fallback and logs only
        `redactSecrets(message)` — never the error object, never React's
        `errorInfo` (a component stack can embed a configured URL). No telemetry,
        no "copy error details" affordance, by design.
      - `redactSecrets()` masks the config segment of any configured manifest URL
        embedded in **arbitrary text** (messages, stacks), keeping the host so a
        redacted log still identifies the addon.
      - The **HTTP-cache half** of the required test exists
        (`http-cache.test.ts`: addon fetches must pass `cache: "no-store"`). The
        **service-worker half is not yet applicable** — there is no service
        worker until P-6 (PWA). It must land with the SW, not before.
- [ ] **No remote UI code (`player`):** UI code is first-party, bundled,
      build-time only — never fetched or `eval`'d at runtime, because it would
      run in the origin holding the debrid credential. **CSS counts as code
      here**: attribute selectors plus `background-image` exfiltrate field
      contents, and restyling can hide the redaction §6a depends on.
      — ARCHITECTURE §6a, §7a
- [ ] **A theme is data, never code (`player`).** Distributable themes are *not*
      a non-goal — installing one is intended (§7b) — but the invariant above is
      what makes it safe, so check the shape rather than the feature:
      - A theme is a flat record of token name → value. No CSS, no JS, no
        selectors. Reject anything that lets a theme express a *rule*.
      - Tokens are applied with `setProperty` **per token**, never by building
        a stylesheet string: `setProperty` parses against the property grammar
        and drops malformed values, so a value carrying `}` cannot escape its
        declaration. This is the second gate behind schema validation.
      - Application iterates the **contract**, not the theme's own keys, so an
        unknown key can never reach the document.
      - Assets, when they land: fonts via `new FontFace(name, buffer)` (no CSS
        string built from theme input); SVG icons allowlist-parsed and rebuilt
        as DOM nodes, never `innerHTML`; textures as size-capped `data:` URIs
        only — an `https:` asset URL would leak the user's IP to the theme's
        author on every launch.
      - The default theme stays reachable, so a bad theme is recoverable.
      — ARCHITECTURE §7a/§7b

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
      is insufficient. **The stamp gate applies to the `QueueItem.resolution`
      cache too, not only the playback FSM (audit A-007)** — a superseded resolve
      landing late must commit nothing (no mutation, no notify), else it poisons
      the memory cache `startItem` reuses with a stale bearer URL. Race tests
      required, asserting **both** FSM state and `QueueItem.resolution`
      (resolve-after-skip, failure-after-success, reorder-during-resolve,
      double-completion, old-success/failure-after-new, current-vs-prefetch). —
      ARCHITECTURE §4b
- [ ] **Queue identity is by stable ID:** `currentItemId` + `playOrder` (not a
      mutable array index); "up next" reads from `playOrder` so it's correct
      under shuffle. Index-as-identity is a defect. — ARCHITECTURE §4a
- [ ] **A queue that holds items is playable (audit A-007):** adding the first
      item to an empty/unselected queue sets a cursor (or `play()` falls back to
      `playOrder[0]`) — "add to queue then press play" must not silently no-op. —
      ARCHITECTURE §4a
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

## 10. Finding calibration — avoid feedback for feedback's sake

Apply these gates before including any audit finding. The goal is to identify
demonstrated failures, not maximize finding count.

- [ ] **Concrete evidence:** reproduce the behavior or establish it directly
      from executable code. Architectural speculation is not a finding.
- [ ] **Present scope:** audit what is implemented and promised in the current
      phase. Do not fail standalone code for a future deployment topology, UI
      integration, or operational model that has not landed.
- [ ] **Observable consequence:** name exactly what breaks for the user, system,
      security boundary, or documented invariant. Optional polish or theoretical
      neatness is not blocking.
- [ ] **Guarantee versus optimization:** distinguish correctness guarantees from
      optional hints and optimizations. Missing an optimization is not a defect
      when the required fallback preserves correctness.
- [ ] **Proportional severity:** high/critical requires a broken explicit
      invariant, unusable primary journey, legal breach, or exploitable security
      failure. Medium requires a reproducible correctness or recovery failure.
      Never inflate severity merely to encourage implementation.
- [ ] **Composition without invented premises:** trace implemented components
      together, but do not assume an undeclared hosting, scaling, or orchestration
      model to manufacture a composition defect.
- [ ] **Minimal useful feedback:** ask whether fixing this now materially reduces
      real risk. If not, record it as a future review consideration or omit it.
- [ ] **Reconsideration is rigor:** when challenged, reassess the requirement and
      concrete impact rather than defending the finding. Remove findings that do
      not survive scrutiny.

## Current status (update as phases land)
**Start here:** the newest-first
[`docs/audits/README.md`](./audits/README.md) registry is authoritative for the
latest audit, supersession, sign-off, and open findings. The prose below is the
chronological history.

**A-012 product implementation re-audit (2026-07-22): changes required — 1
critical.** The A-011 SSRF reconciliation is incomplete: IPv4-mapped IPv6
literals are rejected only in dotted-decimal form. Equivalent hexadecimal
forms such as `::ffff:7f00:1` (loopback) and `::ffff:a9fe:a9fe` (link-local)
are classified as public and handed directly to the socket. The remaining
implemented protocol, legal, player-security, persistence, async-race,
debrid-account-mutation, and present-scope user-journey checks pass. See
[`docs/audits/2026-07-22-product-reaudit.md`](./audits/2026-07-22-product-reaudit.md).

**A-011 Bitbop + browser-security audit (2026-07-21): changes required — 1
critical, 2 medium.** The request-supplied Torznab URL has no outbound-network
policy, so a publicly reachable Bitbop can be induced to fetch loopback,
link-local, or private targets (confirmed SSRF). When every debrid probe fails
transiently, the failures are swallowed and cached as a genuine no-match.
Finally, the configure page offers AllDebrid and uncached mode even though the
former has no adapter and the latter cannot resolve a non-downloaded torrent.
The request-owned credential boundary, no-audio-storage and no-built-in-indexer
legal rules, file selection, resolved HTTPS output, player CSP/redaction, HTTP
no-store behavior, and resolved-media persistence rules pass. See
[`docs/audits/2026-07-21-bitbop-and-browser-security.md`](./audits/2026-07-21-bitbop-and-browser-security.md).

**A-011 (2026-07-21): 1 critical + 2 medium on Bitbop — all reconciled.** The
critical was **SSRF via the caller-supplied indexer URL** — see the new §3 item,
which is now the checklist's most detailed entry because the failure mode has
four distinct parts and a guard can look complete while missing one. Also fixed:
a total debrid outage was swallowed into a cached no-match (now a retryable
uncacheable error, distinguished from a healthy provider's legitimately empty
answer), and the configure page offered AllDebrid + "include uncached" when
neither could produce a stream (both removed from the **schema**, not just the
UI, so an unusable install URL can't be produced or parsed). Bitbop 66 → 122
tests.

**Phase 3 exit criteria MET (2026-07-22) — first live end-to-end run.** The
whole chain ran against a real Real-Debrid account and a real self-hosted
Prowlarr: player → musicmeta → Bitbop → MusicBrainz → Torznab → RD cache check →
`pickFile` → unrestrict → audio. **`pickFile` selected correctly 26/26** across
two independently-encoded rips of one album (13/13 by disc+position each, plus
the fuzzy path). Auditors should note what this run established that no fake
could: **Real-Debrid's file ids are not track order** — in one rip file id 1 was
track 13 and id 13 was track 1 — so any id-order or largest-file assumption
serves the wrong song *confidently*. Audio-only selection held (13 of 14 files,
cover art excluded, `links.length === selected.length`), and cleanup held (two
cached torrents kept, five deliberate misses leaving zero orphans). Two matches
survived only through normalization: `u + me = -3` → `05. u + me = 3.flac`, and a
curly apostrophe (U+2019) in `what’s wrong with me`. Two live-found bugs were
fixed in the same pass — compilations searched for "Various Artists" instead of
the track's own artist, and a slow indexer cost its full timeout on *every*
track because failures weren't remembered (now a replayed cooldown, which
preserves the outage-vs-no-match distinction).

**Debrid account-mutation fix (2026-07-21, not audit-driven).** Reading the
current Real-Debrid API against MediaFusion, Comet, and StremThru surfaced a
defect no fake-based test could have: because `/torrents/instantAvailability`
is gone, `checkCache` was implementing "is this cached?" as
`addMagnet → selectFiles("all") → info` and never cleaning up — so a single
stream request could add **twelve** torrents to the user's account, and
selecting `all` on an uncached one **started downloading an entire album** the
addon had already promised never to wait for. Neither reference implementation
adds anything to check. Fixed per the new §3 item (delete-what-we-added,
audio-only selection, handle threaded check→resolve, non-mutating `GET /torrents`
pre-pass, rationed add-probes) along with RD's in-band `error_code` envelope,
which was being ignored entirely. Bitbop 122 → 140 tests.

**`bitbop` — the `stream-debrid` addon — implemented (2026-07-21).** The
plan's centerpiece (§2/§2a/§3) is built, as `@p2p-songs/bitbop`. 66 tests, all
running against injected fakes (no network, no debrid account). Audit it against
§2 and §3 in full; the highest-value things to attack are (a) whether any code
path can reach a debrid credential that isn't the requesting user's, (b) whether
`pickFile` can return the *wrong track* (Plan §2a) rather than nothing, and
(c) whether any error/diagnostic path can emit key material. Note the addon
ships **no tracker list** — indexers come from config, deliberately. The
player-side **CSP/Trusted Types gate (§7) landed with it**; see §7 for exactly
what to verify and what is honestly out of scope.

**A-010 player P-5 audit (2026-07-21): 1 medium — now reconciled.** The
debounced queue autosave could lose changes made within 800 ms of a close or
reload because no lifecycle flush existed. Investigating it surfaced a larger
version of the same bug: the debounce rescheduled on *any* engine notification,
and the engine notifies ~4×/s on position ticks, so the queue was never
persisted while a track played. **Fixed 2026-07-21** — `SessionAutosave`
(`player/src/core/persistence/session-autosave.ts`) reschedules only on a
changed queue snapshot, flushes the pending snapshot on `visibilitychange`
→hidden / `pagehide` / teardown, and retries a rejected write on the next flush
without spinning. Auditors: the invariant to check is *"a queue edit is durable
by the time the page is hidden, at any playback state"* — a debounce that resets
on playback state is the failure mode. After scope clarification, CSP/Trusted Types
remains a future implementation gate for credential-bearing addons and mobile
layout remains deferred polish outside this lite desktop E2E slice; neither is
a current finding. Protocol/legal neutrality, configured-URL redaction,
resolved-media persistence rules, command/query-plane separation, provider
isolation, and the prior reconciliation fixes pass. All player, SDK/protocol,
and addon tests, typechecks, and builds are green. See
[`docs/audits/2026-07-21-player-p5.md`](./audits/2026-07-21-player-p5.md).

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

**A-006 reconciled (2026-07-20).** All six addressed: (crit) the SDK now detects
the secret-bearing path before any method/OPTIONS early-return, so configured
405/204 are `no-store, private`; (SDK) `meta` route type↔id identity validated
on input (404 on mismatch); `stream-legal` now (a) requires a **recognized
per-item CC/public-domain license** (fail closed; §5 above), (b) requires
**artist agreement** before matching, and (c) distinguishes a **total outage**
(retryable uncacheable error) from a genuine no-match (short cache); and a new
shared **`@p2p-songs/musicbrainz`** package puts MB access behind a **≤1 req/sec
limiter + 503 backoff**, consumed by both addons. New invariants in §5/§6/§6a
above. **SDK 36 tests; addons 46 tests (musicbrainz 7 + musicmeta 14 +
stream-legal 25); typecheck + build + built-package adversarial probes green.**
Re-audit to confirm.

**A-007 player P-1 + A-006 reconciliation audit (2026-07-20): changes required
— 1 high, 1 medium.** P-1's playback reducer drops stale stamps, but the engine
committed stale scheduler outcomes into `QueueItem.resolution` before that check;
an old attempt could overwrite a newer bearer URL. Empty-queue append left no
current item so play was a no-op. A-006 is confirmed. See
[`docs/audits/2026-07-20-player-p1.md`](./audits/2026-07-20-player-p1.md).

**A-007 reconciled (2026-07-20).** Both fixed. (high) The engine now stamp-gates
**every** queue-resolution commit — a per-item `resolutionOp` records the
`attemptId` allowed to write that item, checked immediately before each commit in
`beginResolve`/`prefetchUpcoming`/`tryStream`; a superseded outcome mutates and
notifies nothing (§8 above; ARCHITECTURE §4b). (medium) `append`/`insertAfter`
select the first item when the queue has no cursor, and `play()` falls back to
`playOrder[0]`. Added queue-cache race tests (old-success/failure-after-new,
current-vs-prefetch), empty append/insert/remove-last→append tests, plus two
gap-closures (expiry-freshness reuse, `repeat:"all"` breaker bound). Also
documented P-1's deliberate deferrals in ARCHITECTURE §4b (provider-wide backoff
and the ~30 s re-prefetch net → P-3; consecutive-threshold vs sweep-set). **46
player tests; typecheck + build + built-output probes green.** Re-audit to confirm.

**Player P-3 addon client landed (2026-07-21) — awaiting audit.** The real
addon client (`player/src/core/addon/`) replaces the P-1 fake `Resolver`. New
things an auditor should check against this file: the `AddonClient` **validates
every addon response against the `@p2p-songs/protocol` schema** before it reaches
the engine (§6/§8 — an addon is untrusted input); the `/stream` path stays a
**scheduler-owned command plane** (the resolver adds no retry/cache — §5a/§8);
**provider-wide exponential backoff** now closes the P-1 deferral (§8 "provider
backoff"), and a *reachable* addon returning no match is not counted as a failure
(no false backoff); configured-URL transport is **`https`-only except loopback**
(§6a); the request-URL builder round-trips through the real SDK router in a
**live-HTTP e2e** (`stream-legal` + `musicmeta`, fixture upstreams). The metadata
plane's TanStack Query policy wrapper is intentionally at the app layer (P-5), not
`src/core` — its *absence there* is correct, not drift (ARCHITECTURE §5a). Addon
packages are test-only devDeps; no bundled/runtime addon dependency (§1
neutrality). **88 player tests; typecheck + build + live-HTTP e2e green.**

**A-008 reconciled (2026-07-21).** Both medium findings fixed — failure isolation
at the new external boundary. (1) `AddonCollection.getMeta` isolates a
down/malformed metadata provider and falls through to the next capable addon
(aggregate `AddonUnreachableError` only when none is reachable; cancellation
re-thrown). (2) The stream resolver asks each provider under its own **bounded,
abortable deadline** and aggregates over never-rejecting results, so one hung
addon can't wedge the resolve — a timeout is classified unreachable→backoff,
distinct from a cancellation. New down-first/malformed-first/empty-first/all-down
metadata tests + hung+healthy/all-hung/skip-during-hang resolver tests. **97
player tests; typecheck + build + built-output probes green.** Re-audit to confirm.

**A-008 player P-3 audit (2026-07-21): changes required — 2 medium.** Metadata
fallback stops when the first matching provider throws, so a healthy later
addon is never queried. Stream fan-out has no per-provider deadline and waits
on `Promise.allSettled`, so one hung addon blocks a valid stream already returned
by another. Both failures were reproduced against built output. A-007 is
confirmed; P-3's remaining protocol validation, secret transport, command-plane,
backoff, neutrality, and live happy path pass. See
[`docs/audits/2026-07-21-player-p3.md`](./audits/2026-07-21-player-p3.md).

**Player P-2 real audio subsystem landed (2026-07-21) — awaiting audit.** The
browser audio backend (`player/src/core/audio/`) behind the existing
`AudioBackend` interface. What an auditor should check: **crossfade uses
`element.volume`, never a Web Audio graph** (the CORS-taint reason, ARCHITECTURE
§4c — a `MediaElementAudioSourceNode` on the critical path is a regression);
**events are token-identity gated** so a late `loaded`/`ended`/`error` from a
swapped-away element is dropped, not misapplied (§4b — `ended`/`position` only
from the active element); **dual-element preload→swap** is the gapless mechanism
and the engine preloads only the *immediate* next item (JIT, one idle element,
§5.2); a superseding `load`/`stop` must silence the outgoing element (no audio
bleed); **MediaSession routes OS actions to engine commands and never owns
playback logic** (§8a); a rejected `play()` (autoplay policy) is not treated as a
stream failure. The anticipatory crossfade *trigger* is intentionally deferred to
the position-timing work (§4b/§4c) — its absence is not drift. Audio logic is
unit-tested in node against injected fakes (`MediaElementLike` + fake ticker +
fake session — no jsdom); the Vite harness (`harness/`) is a throwaway dev page
for the manual audible smoke and `vite` is a devDep for it only (no app build
yet). **121 player tests; typecheck + `vite build` green.**

**Player P-4 persistence + catalog fan-out landed (2026-07-21) — awaiting audit.**
Durable state (`player/src/core/persistence/`) and cross-addon catalog search.
What an auditor should check: **persist identity, not resolved media** (§6/§11) —
`saveQueue` must strip every `QueueItem.resolution` and `loadQueue` must rebuild
each item `idle`; a resolved/expiring **bearer URL reaching the store at all is a
defect** (there is a test asserting the raw persisted record contains neither the
URL nor a `resolution` field, through both the in-memory and real-IndexedDB
paths). **Installed addons are secret-bearing (§6a):** they live in their own
table, records carry a `configured` flag, and `redactManifestUrl()` is the only
sanctioned rendering — a raw configured URL in UI, a log, an export, or an error
message is a finding. **Persistence is behind a port** (`PersistenceStore`, §9
decision 2): rules live in `PlayerRepository`, `DexieStore` is a thin adapter, so
engine code binding directly to Dexie would be drift. **Catalog fan-out**
(`AddonCollection.search`) merges across addons **deduped by content id** and must
isolate a down/malformed/**hung** provider — every provider call goes through the
shared `askBounded` helper (`core/addon/fan-out.ts`), which now also backs the
stream resolver and `getMeta`; an unbounded `await`/`Promise.all` over providers
is the A-008 bug class and a regression. Note the repository is **not yet wired to
the engine** (debounced autosave + hydrate-on-boot is explicitly P-5, with the app
shell) — its absence is deferral, not drift.

**A-009 reconciled (2026-07-21).** All 3 medium fixed; the invariants they add:
(1) **A provider deadline must be a hard bound, not a cooperative abort** —
`askBounded` races the task against a timer it owns, so a transport that ignores
its `AbortSignal` still can't wedge a fan-out; merely aborting a child signal and
awaiting the task is the regression to flag, and abandoned tasks must stay
rejection-safe. (2) **Read-modify-write atomicity belongs to the port** —
`PersistenceStore.update` (Dexie `rw` transaction / synchronous memory section)
is the only sanctioned way to mutate a record; a caller composing `get`+`put` is
a defect, because two overlapping playlist edits then silently discard one.
(3) **Play history exists** as an identity-only, retention-capped collection
(`PlayEvent { id, track, playedAt }` — a `TrackRef`, never a resolved stream);
the Dexie schema is declared cumulatively (v1's five tables, v2 adding `history`)
so an existing database upgrades rather than breaking. **166 player tests;
typecheck + build + built-output probes green** (Dexie adapter proven against
`fake-indexeddb`, including reconnect and a real v1→v2 upgrade).

**A-009 player P-4 audit (2026-07-21): changes required — 3 medium.** The
shared `askBounded` deadline remains pending when a task ignores abort;
concurrent playlist read-modify-write operations overwrite one another; and
the phase is marked done without its explicitly promised play-history
collection/API. Queue identity persistence correctly strips bearer stream URLs,
configured addon records remain separated/redacted, catalog merge/dedup works,
and cooperative provider failures are isolated. See
[`docs/audits/2026-07-21-player-p4.md`](./audits/2026-07-21-player-p4.md).

**Player P-5 minimal app slice landed (2026-07-21) — awaiting audit.** A
React/Vite app (`player/index.html` → `src/app/`, `src/ui/`) over the existing
engine. What an auditor should check:
- **Neutrality holds in the UI (§1/§11):** the only way to add a source is
  pasting a manifest URL in the addon manager. No bundled, seeded, suggested, or
  default-installed addon anywhere in `src/app`/`src/ui`. (The *search* screen
  suggests example **queries**, not addons — that is not a bundled source.)
- **Credential display (§6a):** a stored addon URL must only ever render through
  `redactManifestUrl`. A raw configured URL in the DOM, a log, or an error
  message is a finding.
- **The `core`/`ui` boundary (§8):** `src/core` must not import from `src/ui` or
  `src/app`. Components must own no playback/queue logic — they read view-models
  and issue engine commands (§8a).
- **Plane separation (§5a):** the TanStack Query client is app-layer and wraps
  metadata reads only. A `/stream` call routed through React Query — inheriting
  retry/refetch — is a regression.
- **Snapshot stability:** `Engine.getState()` must stay referentially stable
  between real changes; returning a fresh object breaks every diffing subscriber
  (it caused an infinite React render loop). `restoreQueue` must preserve stable
  ids and force `resolution: idle`.
- **Honest failure (§4b, end-to-end value):** a track that can't be resolved must
  say so — `PlaybackAlert` surfaces the resolver's reason and distinguishes "no
  source has this track" from "no stream addon installed" from "addons
  unreachable". Silent failure is the defect this replaced.
- **Not yet built, deliberately** (absence is not drift): router, installable
  themes (§7b — the token contract and three bundled themes landed 2026-07-22;
  install/validate/store did not), source-picker modal, Playlists tab,
  PWA/service worker (P-6). Phase 5's full exit criteria stay open pending
  `stream-debrid` and the measured gapless matrix. **207 player tests;
  typecheck + `vite build` green.**
