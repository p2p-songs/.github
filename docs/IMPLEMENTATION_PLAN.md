# P2P Songs — Implementation Plan

A Stremio-style system for music: a thin **player**, a stateless **addon
protocol** (HTTP+JSON) that anyone can implement, and one self-contained
**debrid-backed stream addon** — the same shape as
[Torrentio](https://github.com/TheBeastLT/torrentio-scraper): discovery
(query indexers/trackers), aggregation (combine and rank those results),
and debrid resolution (turn a chosen result into a direct link) all live
inside that one addon, using debrid services "as it sees fit." Everything
resolves server-side into a direct, Range-servable HTTPS link before the
player ever asks for it. A **discovery/metadata layer** is informed by
[Spotube](https://github.com/krtirtho/spotube)'s metadata-source/audio-source
split, though Spotube itself has no addon protocol or debrid layer (§4).
The player and core never speak BitTorrent — that's the whole point.

---

## 1. How Stremio Actually Works (the part we're copying)

Stremio is three loosely-coupled things that only agree on a wire protocol:

| Layer | Repo | Job |
|---|---|---|
| **Addons** | any HTTP server (SDK: [`stremio-addon-sdk`](https://github.com/Stremio/stremio-addon-sdk)) | Answer `manifest.json`, `/catalog`, `/meta`, `/stream`, `/subtitles` requests with JSON. Stateless, no auth, CORS-open. |
| **Core** | [`stremio-core`](https://github.com/Stremio/stremio-core) (Rust) | The "brain": addon collection, library, search aggregation, player state, settings. `Msg` in, `Effects` out, `Model` updated. Compiled once, reused everywhere. |
| **Player** | closed-source app shell / `stremio-core-web` for the web bridge | Just a `<video>`/MPV element pointed at whatever URL a stream addon returned. |

**Addons never handle media themselves — they hand back a pointer.** That
pointer can be a torrent `infoHash`/`fileIdx`, or it can already be a plain
`url` if the addon resolved it server-side. That second path, taken all the
way, is what Torrentio does — and what we're building.

---

## 2. How `stream-debrid` Works: the Torrentio Pattern

[Torrentio](https://github.com/TheBeastLT/torrentio-scraper) is one
self-contained Stremio addon, not a layer sitting on top of other addons.
On a `/stream` request it:

1. **Discovers** — queries the trackers/indexers it's built to query, for
   candidates matching the requested title.
2. **Aggregates** — combines and ranks those results into one list, inside
   the same addon (no separate meta-aggregator, no fanning out to *other*
   installed addons).
3. **Resolves via debrid, as it sees fit** — if the user pasted a
   Real-Debrid/AllDebrid/etc. key into Torrentio's **own** `/configure`
   page, Torrentio calls that provider's API itself and hands back a direct
   link instead of a torrent pointer. No debrid key configured → it falls
   back to returning a raw `infoHash` for Stremio's local torrent engine.

**`stream-debrid` is that same shape, one to one:** one addon, its own
`/configure` page, its own discovery logic, its own debrid client — no
`StreamProvider` plugin interface for aggregating other addons, no
AIOStreams-style meta-layer. (An earlier draft of this plan added that
plugin interface, reaching for the AIOStreams shape; it's removed.
AIOStreams is a different, separate pattern — set aside for now, per your
call.) Internally that's still two clean, testable modules, just not
exposed as a network-callable boundary:

- **Discovery/aggregation:** `queryIndexers(track) -> Candidate[]` — fans
  out to whatever Jackett/Prowlarr-style indexers are configured, de-dupes
  and ranks the combined results.
- **Resolution:** `resolveViaDebrid(candidate) -> url` — cache-checks and
  calls the configured debrid provider's API, "as it sees fit": cached
  result → resolve immediately; nothing cached → either skip it or (if you
  choose to support it) trigger a download on the debrid side and poll.

`discover -> resolve -> dedupe/sort/label -> respond`, all inside one
addon, one `/configure` page, one deployment.

---

## 3. Legal Compliance Model — Staying in Stremio's Posture

Stremio's legal position rests on a layering that's worth reproducing
exactly, not just gesturing at:

| Layer | Who operates it | Posture |
|---|---|---|
| Protocol + SDK + core + player | Stremio the company | Neutral infrastructure. Bundles only Cinemeta (metadata, no streams). Never hosts, indexes, or endorses a content source. This is why Stremio-the-app is legally comparable to a browser. |
| Torrentio (and similar community stream addons) | Independent, often anonymous third parties — and, notably, **publicly hosted** for anyone to use (torrentio.strem.fun), not just self-hosted | Not affiliated with or endorsed by Stremio; each is its own legal actor. It never stores or caches copyrighted files on its own infrastructure — it returns pointers (magnet hashes, or a debrid link resolved with **the requesting user's own** debrid API key). Public hosting is viable specifically because those two things hold. |

Applying this exactly to our project:

- **`music-addon-sdk` + `music-core` + `player-app` stay fully neutral.**
  They ship with zero bundled stream sources beyond what a user installs by
  pasting a manifest URL. This is the layer that needs to be as clean as
  Stremio-the-app — don't let it default-install `stream-debrid`.
- **`stream-debrid` can be hosted the same way Torrentio is — publicly, if
  you want — as long as the same two invariants hold:** it never stores or
  caches audio files on its own infrastructure (it only ever holds
  candidate metadata: title, hash, size), and every debrid API call uses
  **the requesting user's own** debrid credentials from their `/configure`
  config, never a shared/pooled account it pays for itself. Those two
  invariants, not "must stay self-hosted," are what keep it in Torrentio's
  posture rather than becoming something riskier.
- **The indexer and the debrid account are the user's own.** `stream-debrid`
  can ship built-in discovery logic (Torrentio does too — it doesn't make
  users bring their own indexer), but the debrid subscription is always the
  end user's, entered per-request via config, never baked into the addon.
- **One caveat worth flagging plainly (not legal advice):** debrid services
  themselves have had mixed legal outcomes in different jurisdictions
  (e.g. rulings against some providers in France) even though the
  "personal cyberlocker" model is broadly how they justify operating. This
  varies by service and country — it's a real caveat, not a solved
  question, and it's on the user operating the tool, same as it is for
  every Stremio+debrid user today.
- **`stream-legal` remains the always-uncontroversial path** — build and
  demo on it first, exactly as before.

---

## 4. Discovery & Metadata — What Spotube Confirms and Adds

**Scope check first, since it's easy to overstate:** Spotube has **no
equivalent of Stremio's addon protocol, and no torrent/debrid layer at
all.** Its "plugins" are audio-source *backends compiled into or loaded by
the app itself* (YouTube Music via yt-dlp/Piped/Invidious, or Spotify's own
stream via a Premium account) — not independently-hosted HTTP/JSON services
with a `manifest.json` that any third party can implement and the player
calls over the network. There's no `catalog`/`stream` resource split, no
`/configure`-style credential handoff between separately-run services, and
nothing torrent- or debrid-shaped anywhere in it. So on the specific thing
this project is about — a network addon protocol, and a debrid-backed
stream addon — Spotube isn't a precedent at all; that part is entirely
Stremio/Torrentio territory (§2). What Spotube *is* a precedent for is
narrower: it independently converged on the same **metadata-source /
audio-source separation** we're already using, and it validates the
specific metadata APIs to use. Details below.

[Spotube](https://github.com/krtirtho/spotube) is an independently-built
system that converged on the *same split* we're already using — metadata
source and audio source are separate, swappable plugins ("bring your own
music metadata/playlist/audio-source with plugins"). That's a strong
outside validation of the meta-addon/stream-addon separation, not a reason
to change the architecture. What it does contribute:

1. **It confirms our metadata stack, not just resembles it.** Spotube's
   credited data sources are MusicBrainz, ListenBrainz, and LRCLib — the
   exact same three we picked independently for `musicmeta`/`lyrics-lrclib`.
   Worth adding: **ListenBrainz** (the MusicBrainz Foundation's fully open
   scrobbling/recommendation database — no API key, no ToS risk) as a
   *second* data source inside `catalog-charts`, for "trending"/"similar
   artists"/personalized-ish charts that MusicBrainz's raw metadata graph
   doesn't give you. This is the free, always-safe upgrade to catalog
   richness — do this before anything Spotify-shaped.
2. **It validates a `stream-ytmusic` addon** as a third stream-source tier,
   distinct from `stream-legal` and `stream-debrid`. Spotube's own audio
   plugins lean on YouTube-adjacent tooling (`yt-dlp`, Piped, Invidious,
   NewPipeExtractor) precisely because YouTube has near-complete music
   catalog coverage with no cost. For **our** compliance bar, prefer the
   cleaner half of that idea: mirror Stremio's own native `ytId` stream
   field (*"plays using the built-in YouTube player"*) — i.e. hand back a
   YouTube video ID and let the player embed YouTube's own official
   IFrame/Music player, rather than extracting raw audio streams with
   `yt-dlp`. The former is an explicitly-supported playback mode in
   Stremio's own protocol; the latter is what Spotube/Piped do but sits on
   shakier ToS ground. Build the `ytId`-style version; treat raw extraction
   as a documented-but-not-default option if you ever want it.
3. **An optional, explicitly grayer stretch:** a Spotify-metadata addon
   (search, personalized recommendations, playlists) for richer discovery
   than MusicBrainz/ListenBrainz alone can give you. Spotify's official Web
   API supports app-only (client-credentials) auth for catalog
   search/browse with no per-user login — but scraping or using
   undocumented endpoints (which some Spotify-metadata-only tools have
   historically done to avoid Developer-account friction) is explicitly
   against Spotify's ToS. Keep this optional and clearly labeled greyer
   than everything else in the stack, same tiering logic as
   `stream-legal` vs `stream-debrid`.

---

## 5. Concept Mapping: Video → Audio

| Stremio (video) | This project (audio) | Notes |
|---|---|---|
| `movie` / `series` types | `artist` / `album` / `track` / `playlist` types | Track is the atomic streamable unit, like an episode. |
| IMDb ID (`tt1234567`) namespace | **MusicBrainz ID (MBID)** primary, `isrc:` secondary idPrefix | Open, canonical, freely-reusable — the direct IMDb equivalent. |
| Cinemeta (official meta addon) | **`musicmeta`** — MusicBrainz + Cover Art Archive | Same role: canonical metadata for anything named by ID. |
| Catalog addon (Top IMDb, etc.) | `catalog-charts` — MusicBrainz browse + **ListenBrainz** trending/similar-artist data | ListenBrainz addition confirmed by Spotube (§4). |
| **Torrentio** (single addon: scrape many trackers + own debrid resolve) | **`stream-debrid`** — single addon: query many indexers + own debrid resolve | Primary stream source, and the addon shape we're actually copying — see §2. |
| Subtitles resource | **Lyrics resource** — same shape, `{ id, url, lang }` | Synced `.lrc` via LRCLib if available. |
| Built-in YouTube player (`ytId` stream field) | **`stream-ytmusic`** — same `ytId`-style field, official embed | New addon, added per §4. |
| Local BitTorrent streaming server | **Cut from the critical path** | See Phase 8 — optional, lowest priority. |
| `<video>` / MPV player | `<audio>` element + queue/gapless controller | Needs `MediaSession` for lock-screen controls, which video mostly doesn't. |
| stremio-core (Rust state machine) | **`music-core`** — TS first, optional Rust/WASM port later | Deliberate divergence — see §6. |

---

## 6. Architecture Diagram

```mermaid
flowchart LR
    subgraph Client["Player App (web)"]
        UI[React UI: library, search, now-playing]
        Core[music-core state machine]
        AudioCtl["Audio Controller — audio tag + queue + YouTube embed"]
        UI <--> Core
        Core --> AudioCtl
    end

    Core -- "manifest.json / catalog / meta / stream / lyrics" --> Meta[musicmeta addon]
    Core --> Catalog["catalog-charts addon\n(MusicBrainz + ListenBrainz)"]
    Core --> StreamLegal[stream-legal addon\nJamendo / Internet Archive / FMA]
    Core --> StreamYT["stream-ytmusic addon\n(ytId, official embed)"]
    Core --> StreamDebrid["stream-debrid addon\n(Torrentio-style: self-contained)"]
    Core --> Lyrics[lyrics-lrclib addon]

    AudioCtl -- "direct HTTPS URL" --> StreamLegal
    AudioCtl -- "ytId -> official YouTube embed" --> StreamYT
    AudioCtl -- "direct HTTPS URL" --> StreamDebrid

    subgraph StreamDebridInternal["inside stream-debrid — one addon, one /configure page"]
        Discover["discovery/aggregation:\nqueryIndexers(track) -> Candidate[]\n(fans out to N indexers/trackers,\ndedupes + ranks internally)"]
        DebridClient["debrid resolution:\nresolveViaDebrid(candidate) -> url\n(Real-Debrid / AllDebrid / Premiumize / TorBox)"]
        Discover -- "candidates: infoHash" --> DebridClient
        DebridClient -- "cache-check + resolve -> url" --> Rank["dedupe / sort / label"]
    end
    StreamDebrid --> Discover
    Rank --> StreamDebrid
    DebridClient -. "torrent downloaded on debrid's\nservers — never touches our infra" .-> Swarm((BitTorrent swarm))
```

The player and `music-core` never see a torrent, an `infoHash`, or a
BitTorrent client — everything inside the `stream-debrid` subgraph is that
addon's own internal implementation detail.

---

## 7. Tech Stack Decisions

| Area | Decision | Why |
|---|---|---|
| Language | TypeScript everywhere except the optional Rust stretch phase | One language across SDK, addons, core, player. |
| Monorepo | pnpm workspaces (+ Turborepo) | Addons, SDK, core, player app live together. |
| Addon transport | Reuse Stremio's protocol almost verbatim | Proven, minimal — the real work is audio-specific resources and the aggregator. |
| ID namespace | MusicBrainz ID (MBID) primary, ISRC secondary `idPrefix` | Open, canonical, IMDb-equivalent. |
| Metadata source | MusicBrainz API + Cover Art Archive + **ListenBrainz** | Confirmed by Spotube's own stack (§4) — all three are free, no-API-key, ToS-clean. |
| Primary stream source | **`stream-debrid`** — one self-contained addon, the Torrentio shape (§2) | Discovery (query indexers), aggregation (combine/rank results), and debrid resolution all live inside this one addon with its own `/configure` page. |
| Legal/no-debrid demo path | `stream-legal` (Jamendo, FMA, Internet Archive) + **`stream-ytmusic`** (`ytId`-style official embed) | Two tiers of "safe": outright CC-licensed direct files, and YouTube's own explicitly-supported embed mode (mirrors Stremio's native `ytId` field). Build both before `stream-debrid`. |
| Discovery (finding candidate releases) | Jackett/Prowlarr-style indexer proxy, built directly into `stream-debrid`'s own discovery module | Same as Torrentio: built-in scraping logic, not a pluggable external source. |
| Debrid integration | Real-Debrid first, then AllDebrid/Premiumize/TorBox behind a shared `debrid-clients` interface (`checkCache`, `resolve`) | Matches Torrentio's own "pick your provider in `/configure`, works across all of them" model. |
| Addon configuration | `/configure` HTML page → config JSON/base64-encoded into the manifest URL path | Exactly Stremio's `configurable` `behaviorHints` pattern and how Torrentio's own configure page works. |
| Core state pattern | Elm-style `Msg → Effects → Model` | Modeled on stremio-core's runtime, reimplemented in TS to learn the pattern without Rust/WASM cost up front. |
| Player app | React + Vite | Fast dev loop, matches stremio-web conceptually. |
| Playback | HTML5 `<audio>` for direct URLs, YouTube IFrame embed for `stream-ytmusic`, `MediaSession API` for lock-screen/media-key control | Every non-YouTube stream is already a plain Range-servable HTTPS URL by the time the player sees it. |
| Storage | IndexedDB (`idb`) for library/addon collection | Debrid API key lives only inside the `stream-debrid` addon's own config, never in the player. |
| Streaming server | **Cut from MVP** — Phase 8 stretch only | Debrid links are already direct HTTPS. |
| Operating model | `stream-debrid` never stores audio itself and always resolves debrid using the requesting user's own credentials from `/configure` — never a shared/pooled account; protocol/SDK/core/player stay neutral and never bundle it by default | See §3 — this is the legal-compliance decision, not just an infra one. Public hosting (à la Torrentio) is fine as long as those invariants hold. |
| Addon hosting | Stateless addons (`musicmeta`, `catalog-charts`, `stream-legal`, `stream-ytmusic`, `lyrics-lrclib`) → Cloudflare Workers/Vercel functions. `stream-debrid` → small always-on Node service (outbound calls to indexers + debrid APIs need a persistent process, not a functions runtime). | Same shape as how Torrentio itself is deployed. |

---

## 8. Addon Protocol Spec (music version)

**Content types:** `artist`, `album`, `track`, `playlist`

**Resources:** `catalog`, `meta`, `stream`, `lyrics`

**ID scheme:**
- Canonical: `mbid:<musicbrainz-uuid>` for artist/album/track
- Album tracks: `mbid:<release-mbid>:<track-number>`, mirroring Stremio's
  `tt1234567:1:2` (imdbId:season:episode).

**Stream object** — three shapes in practice, one per addon tier:

```jsonc
// stream-legal / stream-debrid: fully resolved, direct
{ "url": "https://…/track.flac", "name": "FLAC · cached",
  "behaviorHints": { "bingeGroup": "album:<release-mbid>", "filename": "…" } }

// stream-ytmusic: official embed, mirrors Stremio's native ytId field
{ "ytId": "dQw4w9WgXcQ", "name": "YouTube Music" }
```

The protocol still technically supports a torrent pointer
(`infoHash`/`fileIdx`) for parity with Stremio and for the optional Phase 8
fallback, but no reference addon emits one — `stream-debrid` always resolves
before responding.

**Lyrics object:**

```jsonc
{ "lyrics": [{ "id": "lrclib-123", "lang": "eng", "url": "https://…/synced.lrc" }] }
```

---

## 9. Repo Layout

```
p2p-songs/
  packages/
    music-addon-sdk/       # analog of stremio-addon-sdk
    music-core/            # Elm-style state machine (Msg/Effect/Model)
    debrid-clients/        # shared debrid API adapters: Real-Debrid, AllDebrid, Premiumize, TorBox
    player-app/            # React/Vite web player
  addons/
    musicmeta/             # MusicBrainz + Cover Art Archive
    catalog-charts/        # MusicBrainz browse + ListenBrainz trending/similar
    stream-legal/          # Jamendo / Internet Archive / FMA
    stream-ytmusic/        # ytId-style, official YouTube embed
    stream-debrid/         # Torrentio-style: discovery + aggregation + debrid resolution, one addon
      discovery/            # queries multiple indexers/trackers, dedupes + ranks
      debrid/               # -> uses packages/debrid-clients
    lyrics-lrclib/          # lyrics via lrclib.net
  docs/
    IMPLEMENTATION_PLAN.md
```

---

## 10. Phases

### Phase 0 — Study (no code)
Read the addon protocol docs (`stremio-addon-sdk/docs/api/`), skim
`stremio-core`'s Msg/Effect pattern, install Torrentio into a real Stremio
client and look at its own `/configure` page (debrid key entry, indexer/
quality options) — that's the actual UX spec for `stream-debrid`'s
`/configure` page (§2) — and skim [Spotube](https://github.com/krtirtho/spotube)'s
plugin docs to see its metadata/audio-source split first-hand (note: it has
no addon protocol or debrid layer of its own, §4).

### Phase 1 — Protocol Definition
Write `docs/PROTOCOL.md`: manifest schema, resource/type list, ID scheme,
stream/lyrics object shapes (§8 is the draft — lock it in as `v0.1`).

**Exit criteria:** hand-written `manifest.json` + one hand-written
`/stream/track/mbid:...json` response, validated against your own schema.

### Phase 2 — `music-addon-sdk`
`addonBuilder()`, `defineCatalogHandler`/`defineMetaHandler`/
`defineStreamHandler`/`defineLyricsHandler`, CORS'd Express router,
`serveHTTP()`, and a `/configure` route helper (config read back out of the
URL path on every request) since `stream-debrid` needs it from day one.

**Exit criteria:** a "hello world" addon in <20 lines; a "hello
configurable world" addon whose manifest URL embeds a config value.

### Phase 3 — Reference Addons
1. `musicmeta` — MBID → metadata + cover art
2. `catalog-charts` — MusicBrainz + ListenBrainz-backed catalogs
3. `stream-legal` — Jamendo/Internet Archive direct-URL streams
4. `stream-ytmusic` — `ytId`-style official YouTube embed
5. `lyrics-lrclib` — lyrics resource
6. **`stream-debrid`** — one self-contained addon, Torrentio's shape (§2):
   - `/configure` page: debrid provider + API key, indexer selection
   - Discovery/aggregation: query the configured indexers for candidates,
     dedupe and rank the combined results internally
   - Debrid resolution: cache-check + resolve any `infoHash` candidate to
     a direct `url`, using the API key from `/configure`
   - Sort/label final results by cached-status, format, source

**Exit criteria:** each manifest installs and responds correctly via
`curl`; `stream-debrid` returns a playable direct URL for a real album
given a real debrid API key.

### Phase 4 — `music-core`
Addon collection (install by manifest URL), aggregated search/catalog,
library/favorites, player state, settings. `dispatch(msg) -> effects[] ->
runEffects() -> new model -> notify subscribers`.

**Exit criteria:** headless — a Node script installs `stream-debrid`,
browses a catalog, resolves a track to a playable URL, zero UI.

### Phase 5 — Player App
Addon manager (install by manifest URL), search/browse, now-playing bar,
queue view. `<audio>` wrapper with gapless crossfade via `bingeGroup`,
YouTube IFrame embed fallback for `stream-ytmusic`, `MediaSession API`.

**Exit criteria:** install `stream-debrid` + `musicmeta`, browse, play a
full album gapless off direct debrid links, control it from OS media keys.

### Phase 6 (stretch) — Rust/WASM Core Port
Port `music-core` to Rust/WASM, mirroring `stremio-core-web`'s `env` trait
bridge pattern.

### Phase 7 (stretch) — Native Shell / Casting
Tauri/Electron shell; Chromecast/AirPlay once the web player is solid.

### Phase 8 (optional, lowest priority) — Local Torrent Fallback
A `webtorrent`-based Node service for uncached-torrent/no-debrid fallback,
invoked only when a stream addon returns an `infoHash`. Not part of the
main build order.

---

## 11. References

- Addon SDK & protocol docs: https://github.com/Stremio/stremio-addon-sdk (see `docs/api/`)
- Core state machine: https://github.com/Stremio/stremio-core
- WASM web bridge: https://github.com/Stremio/stremio-core/tree/development/stremio-core-web
- Torrentio (the addon shape `stream-debrid` is modeled on — single addon, own scraping, own debrid resolve): https://github.com/TheBeastLT/torrentio-scraper
- AIOStreams (a different, meta-aggregator pattern — set aside for now, not part of the current plan): https://aiostreams.elfhosted.com/stremio/configure
- Spotube (metadata-source/audio-source split precedent — no addon protocol or debrid layer of its own, see §4): https://github.com/krtirtho/spotube
- MusicBrainz API: https://musicbrainz.org/doc/MusicBrainz_API
- Cover Art Archive: https://musicbrainz.org/doc/Cover_Art_Archive/API
- ListenBrainz API: https://listenbrainz.org/api-docs/
- lrclib (synced lyrics API): https://lrclib.net
