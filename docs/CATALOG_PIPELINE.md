# The curated catalog pipeline (metadata plane)

How the music catalogue that powers search is built, stored, and served. This is
the authoritative design for the **metadata plane**; `DEPLOYMENT.md` covers where
the pieces run, and `IMPLEMENTATION_PLAN.md` §Phase-3 covers `musicmeta` the addon.

## The shift: accelerator → curated store

The metadata plane began as an *accelerator* — Meilisearch in front of a live
MusicBrainz, read-through / write-back, "never a dependency." That design had two
flaws that showed up the moment it met real queries:

- **Junk.** Writing back whatever a free-text MusicBrainz search returned filled
  the index with parodies, covers and bootlegs. "justin bieber baby" surfaced
  "Justin Bieber's Black Baby" by Murs; the actual song was not even in
  MusicBrainz's result set for that query.
- **Runtime coupling to MusicBrainz** — every cold query paid the ≤1 req/sec/IP
  budget, which does not scale to many players.

So the plane was inverted. **Meilisearch is now the curated catalogue that
`musicmeta` serves from, and MusicBrainz is an *offline* source for building it —
never touched at request time.** This is "Spotify/Billboard-grade": precise,
official-only, popular-scoped.

Consequence, stated plainly: **Meili is now a required store, not an optional
accelerator.** If it is empty, the catalogue is empty — there is no live-MB
fallback. That raises the resilience stakes, which is exactly why the golden
dataset (below) lives in storage we own and the index is rebuildable.

What did **not** change: the catalogue is still **identity-only** (artist / album
/ song names, ids, posters — no hashes, no stream sources), so it stays legally
inert and shareable, and it remains the one plane a hosted player may
default-install (neutrality governs the *stream* plane; see REVIEW_CHECKLIST §1).

## Shape

```
  OFFLINE (off the serving box — nightly GitHub Action)
  ┌───────────────────────────────────────────────────────────┐
  │  ListenBrainz popularity ─┐                                │
  │                           ├─► build ─► curated NDJSON ─► publish
  │  MusicBrainz canonical ───┘   (official-only)              │ │
  │  bulk dump (CC0)                                           │ │
  └───────────────────────────────────────────────────────────┘ │
                                                                  ▼
                              ┌─────────────────────────────────────┐
                              │  R2 (object storage WE own)          │
                              │  datasets/catalog-<ts>.ndjson (immutable)
                              │  latest.json  (sha256 + per-type counts)
                              └─────────────────────────────────────┘
                                                                  │
  RUNTIME (inside Railway — can reach private Meili)              ▼
  ┌───────────────────────────────────────────────────────────────┐
  │  import ─► verify sha256 ─► zero-downtime reindex ─► Meilisearch │
  │                                                          │       │
  │                              musicmeta ◄─── serves ──────┘       │
  │                                  │  unified search (Meili only)  │
  │                                  ▼                               │
  │                               player                            │
  └───────────────────────────────────────────────────────────────┘
```

The **golden dataset in R2 is the system of record.** Meilisearch and the compute
host (Railway) are disposable: restore or migrate providers by re-importing the
NDJSON. R2 is the public handoff between the offline and runtime halves.

## Curation — official only, by construction

Two CC0 sources are **joined**, popularity from one and content from the other:

- **Popularity scope** — ListenBrainz **sitewide top artists** (`/1/stats/sitewide/
  artists`): the most-listened artists, each with a real all-time listen count. The
  top ~1000 artists is billboard-grade scope *without* Billboard's proprietary chart
  data (which we cannot redistribute). (The sitewide stats hard-cap at ~1000 rows per
  type, which is why we scope at the *artist* level and take breadth from the dump,
  rather than trying to page millions of recordings.)
- **Content/breadth** — the **MusicBrainz canonical data dump**
  (`canonical_musicbrainz_data.csv`, CC0): deduplicated (one canonical release per
  recording, which dissolves the edition/pressing ambiguity that bit the old design),
  streamed offline. The build keeps only the rows whose **primary artist MBID is in
  the popularity scope**, so each popular artist's whole catalogue enters with no
  per-artist API calls.

Each document's base ranking `score` is its **artist's listen count**, so a top
artist's songs outrank a #900 artist's on a relevance tie. On top of that, a track
that is itself among ListenBrainz's **top recordings** gets that song's listen count
added — the per-song boost that floats the studio hit above the same artist's own
live / acoustic / remix versions (the canonical dump keeps all of them, and within
one artist they would otherwise be score-tied). Only the very top songs are boosted
(the sitewide endpoint caps at ~1000), which is exactly the high-traffic search set.

> **A correction worth recording.** An earlier build used the canonical dump's own
> `score` column as the popularity signal. It is **not** one — it behaves like a row
> ordinal, so "top-N by score" returned ~random obscure recordings (a live check
> found no popular artist in the index at all). The ListenBrainz listen-count join
> above is the fix, and it is what the plan meant by "ListenBrainz popularity **+**
> MB canonical dump" all along.

**Official by construction:** scoping to popular artists excludes parodies/covers by
*other* people entirely (a Weird Al parody has a different artist MBID), and the build
never runs a free-text search. The remaining looseness — a popular artist's own live/
remix recordings can appear — is acceptable (they are legitimately that artist's) and
far milder than the old free-text junk.

Only the ListenBrainz artist list is fetched over the network (a few fast, uncapped
calls); the multi-GB canonical dump is streamed offline, so a full rebuild is fast and
never hits the MusicBrainz ≤1 req/sec budget.

## The golden dataset & versioning

Each build is written to R2 as an **immutable, timestamped** object under
`datasets/`, and one `latest.json` manifest points at the current one. We version
with dated keys deliberately — *not* the bucket's native object versioning —
because immutable, human-readable snapshots are simpler to reason about and the
scheme is **portable to any S3-compatible provider** (B2, S3, MinIO). The engine
and the provider stay disposable; the data is what we own.

The manifest carries:
- **`sha256`** of the NDJSON — `fetchLatest` verifies it before anything is
  indexed, so a truncated/corrupt upload can never reach the live catalogue.
- **per-type `counts`** (`{artist, album, track}`) — surfaced by the addon so the
  player can show "X songs · Y albums · Z artists indexed" and set the right
  expectation about catalogue scope (it is curated, not all of recorded music).

Rollback = repoint `latest.json` at an older snapshot. Retention = an R2 lifecycle
rule expiring old `datasets/*`.

## Zero-downtime reindex

`import` never mutates the live index in place. It builds a fresh
`<index>__staging`, applies the validated search settings, streams the NDJSON in
(**in `batchDocs`-sized chunks** — the full catalogue is hundreds of MB and Meili
caps a single upload at ~95 MiB, so one payload would 413), then **atomically
swaps** staging with the live index and drops staging. Two properties fall out:
readers see the complete old catalogue until the instant the complete new one is
ready, and **removals propagate** — a song dropped from the curated set actually
disappears (a merge-in-place would leave it behind).

## Unified search & ranking

The player has **one search box** over artists, albums and songs — not a
per-type search. A single Meili query (no type filter) returns all three,
relevance-ranked together: "justin bieber" → the artist, "my world" → the album,
"baby" → the song.

Ranking is driven by a stored **`searchtext = "<artist> <album> <title>"`** field
(searchable-attributes: `searchtext`, then `name`, `description`; filterable:
`type`; sortable: `score`). Popularity is the **final ranking tiebreaker**: the
ranking rules end in `score:desc`, so relevance still decides first but among
equally-relevant hits the more popular one wins (the `score` is the artist's
ListenBrainz listen count). Putting artist + album + title adjacent in one field is
what makes real user queries work, validated against Meili's ranking with real data:

| A user types… | resolves to |
|---|---|
| `baby` | the song **Baby** |
| `baby justin bieber` / `justin bieber baby` (any order) | the song **Baby** |
| `my world baby` (album + song) | the song **Baby** |
| `justin bieber` | the **artist** |
| `my world` | the **album** My World 2.0 |
| `weekend blinding lights` (typo) | **Blinding Lights** (typo-corrected) |

## Invariants

- **Identity only** — no hashes, no stream sources, ever. Legally inert; the
  metadata plane stays shareable and default-installable. (Unchanged.)
- **Neutrality** — a default *metadata* addon is allowed; the *stream* plane
  (Bitbop) stays strictly user-installed with no bundled sources or credentials.
  (Unchanged — REVIEW_CHECKLIST §1.)
- **Meili is a required curated store** (changed from "accelerator, never a
  dependency"). Its resilience is provided by the golden dataset, not by a
  live-MB fallback.
- **Curated, not exhaustive** — the catalogue is deliberately scoped to popular,
  official music. The player communicates this via the indexed counts.

## Resilience / disaster recovery

- **System of record** = the versioned NDJSON in R2, independent of Railway.
- **Meili/Railway die** → spin Meili up on any provider, `import` from R2, live in
  minutes. Nothing to back up on the compute host.
- **R2 lost** → the pipeline (seed + build code) is in git and the sources are
  CC0, so the whole dataset is regenerable from scratch. Nothing is unrecoverable.
- Snapshot Meili → R2 too (a fast-restore convenience), but the NDJSON is the real
  safety net. Railway's lack of managed backups is therefore irrelevant.

## Where it runs

- **build + publish** — off the serving box (nightly **GitHub Action**), since it
  processes multi-GB dumps and only needs R2 credentials.
- **import** — **inside Railway**, because Meili is private
  (`meilisearch.railway.internal`); a scheduled job (or `musicmeta` noticing
  `latest.json`'s sha256 changed) fetches from R2 and reindexes.
- **R2** — the public handoff between the two halves.

## Implementation

The offline pipeline is the **`@p2p-songs/catalog-builder`** package in the
`addons` repo (not shipped in the runtime addon) — `build | publish | stage |
import | fetch | versions | rollback`, credentials from the environment only. See
its `README.md`.

**The data source, concretely.** `build` first fetches the top `CATALOG_ARTIST_LIMIT`
artists (default 1000) from ListenBrainz sitewide stats — a few fast calls giving
`{mbid, name, listenCount}` each — then streams the **MusicBrainz canonical data
dump** (`canonical_musicbrainz_data.csv`; CC0; ~2 GB zstd, refreshed the 1st &
15th), keeping only rows whose primary artist MBID is in that set. It derives artist
docs from the ListenBrainz list (clean names + listen counts) and album/track docs
from the kept canonical rows, deduplicated per entity, each carrying its artist's
listen count as the ranking `score` and a Cover Art Archive poster from its canonical
release. Memory is bounded to the scoped artists' catalogues, not the whole dump.

**Status (2026-07-24):** storage + import proven end-to-end against real R2 and
Meili (versioned publish, checksum-verified fetch, zero-downtime import, unified
search over the imported data). Built and pending:

- [x] R2 versioned golden-dataset layer (publish/fetch/verify/versions/rollback)
- [x] Zero-downtime Meili import (staging + atomic swap)
- [x] `searchtext` ranking + unified cross-type search (validated)
- [x] Popularity (`score` = artist listen count) as the final ranking tiebreaker
- [x] Per-type counts (from Meili's `type` facet, served at `/stats`)
- [x] **Full-scale source** — `build` = ListenBrainz top artists ⋈ MB canonical dump
      (real popularity scope; replaces both the `build-sample.mjs` prototype and the
      abandoned canonical-`score` attempt)
- [x] Nightly GitHub Action (build+publish → R2)
- [x] Railway import job (scheduled `import` inside the private network) —
      `deploy/catalog-importer.Dockerfile` + compose `import` profile + railway docs
- [x] Slim `musicmeta` to Meili-only search + a `/stats` endpoint (counts) —
      read-only Meili client, no read-through/write-back, `SDK serveHTTP extraRoutes`
- [x] Batched import (Meili caps a payload at ~95 MiB; the full catalogue is
      hundreds of MB, so `import` uploads in `batchDocs`-sized chunks)
- [x] Player: unified search (one box, sectioned) + "X songs · Y albums · Z artists
      indexed" awareness line, from `/stats`

**Live figures (2026-07-24, top-1000-artist scope):** ~902k docs — 993 artists,
144k albums, 756k tracks. Spot-checked search: Taylor Swift / Radiohead / Queen /
Daft Punk (typo-tolerant) all resolve correctly; "my world baby" → Baby by Justin
Bieber. Within-artist version disambiguation is handled by the per-song boost.
