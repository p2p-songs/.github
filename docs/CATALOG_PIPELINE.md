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

- **Content + scope in one file** — the **MusicBrainz canonical data dump**
  (`canonical_musicbrainz_data.csv`, CC0). It is deduplicated (one canonical
  release per recording, which dissolves the edition/pressing ambiguity that bit
  the old design) *and* carries a `score` column that is **ListenBrainz-listen-
  derived popularity**. Keeping the **top-N by score** gives billboard-grade
  scope *without* Billboard's proprietary chart data (which we cannot
  redistribute) and without a separate popularity join.
- **Official by construction** — the canonical mapping already prefers official
  releases over compilations/live/bootleg when it picks the canonical release, and
  the build **never runs a free-text search** (which is what surfaced parodies in
  the old design). So junk cannot enter through the query path. The one
  approximation versus the API prototype: the strict release-group secondary-type
  filter (`-secondarytype:*`) isn't expressible from the canonical CSV alone — the
  popularity scope and canonical selection stand in for it.

Nothing is built by crawling the MusicBrainz API at ≤1 req/sec; the bulk dump is
streamed and processed offline, so a full rebuild is fast and never rate-limited.

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
`<index>__staging`, applies the validated search settings, streams the NDJSON in,
then **atomically swaps** staging with the live index and drops staging. Two
properties fall out: readers see the complete old catalogue until the instant the
complete new one is ready, and **removals propagate** — a song dropped from the
curated set actually disappears (a merge-in-place would leave it behind).

## Unified search & ranking

The player has **one search box** over artists, albums and songs — not a
per-type search. A single Meili query (no type filter) returns all three,
relevance-ranked together: "justin bieber" → the artist, "my world" → the album,
"baby" → the song.

Ranking is driven by a stored **`searchtext = "<artist> <album> <title>"`** field
(searchable-attributes: `searchtext`, then `name`, `description`; filterable:
`type`; sortable: `score`). Popularity is the **final ranking tiebreaker**: the
ranking rules end in `score:desc`, so relevance still decides first but among
equally-relevant hits the more popular one wins (the `score` is the canonical
dump's ListenBrainz-derived popularity). Putting artist + album + title adjacent in
one field is what makes real user queries work, validated against Meili's ranking
with real data:

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

**The data source, concretely.** `build` streams the **MusicBrainz canonical data
dump** (`canonical_musicbrainz_data.csv`; CC0; ~2 GB zstd, refreshed the 1st &
15th). That one file is already deduplicated (one canonical release per recording)
*and* carries a `score` column that is a ListenBrainz-listen-derived popularity
ranking — so it supplies both the content and the popularity scope in a single
pass, no separate join and no live API. We keep the **top-N recordings by score**
(`CATALOG_LIMIT`, default 250 000) in a bounded min-heap and derive artist/album/
track docs from exactly that set (artist docs only for single-artist credits — a
joint "X feat. Y" credit has no single name to attribute). Official-by-construction:
canonical selection already prefers official releases and no free-text search is
ever run, so parodies/covers/bootlegs can't enter. The stricter release-group
secondary-type filter (`-secondarytype:*`) that the API traversal applied is not
expressible from the canonical CSV alone; the popularity scope + canonical selection
stand in for it, and it's the one deliberate approximation versus the prototype.

**Status (2026-07-24):** storage + import proven end-to-end against real R2 and
Meili (versioned publish, checksum-verified fetch, zero-downtime import, unified
search over the imported data). Built and pending:

- [x] R2 versioned golden-dataset layer (publish/fetch/verify/versions/rollback)
- [x] Zero-downtime Meili import (staging + atomic swap)
- [x] `searchtext` ranking + unified cross-type search (validated)
- [x] Popularity (`score`) as the final ranking tiebreaker
- [x] Per-type counts in the manifest (for the player)
- [x] **Full-scale source** — `build` over the MB canonical dump (popularity-scoped,
      official-by-construction; replaces the `build-sample.mjs` prototype)
- [x] Nightly GitHub Action (build+publish → R2)
- [ ] Railway import job (scheduled `import` inside the private network)
- [ ] Slim `musicmeta` to Meili-only serving + a `/stats` endpoint (counts)
- [ ] Player: unified search UI + "X songs · Y albums · Z artists indexed"
