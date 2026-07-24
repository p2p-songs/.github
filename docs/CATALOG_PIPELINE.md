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

- **Scope** comes from **ListenBrainz popularity** (CC0 listen data — which
  artists/recordings are actually popular). Billboard-grade curation *without*
  Billboard's proprietary chart data (which we cannot redistribute).
- **Content** comes from the **MusicBrainz canonical bulk dump** (CC0; already
  NDJSON; deduplicated — one canonical record per recording/release, which also
  dissolves the edition/pressing ambiguity that bit the old design).
- **Official-only filter:** albums are official studio release-groups
  (`primarytype:album AND -secondarytype:*`); traversal is *by artist*, never
  free-text — so parodies/live/compilations/bootlegs can never enter. This is the
  same filter the shared MusicBrainz client's `artistDiscography` already applies.

Nothing is built by crawling the MusicBrainz API at ≤1 req/sec; the bulk dumps are
processed offline, so a full rebuild is fast and never rate-limited.

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
`type`). Putting artist + album + title adjacent in one field is what makes real
user queries work, validated against Meili's ranking with real data:

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
`addons` repo (not shipped in the runtime addon) — `publish | stage | import |
fetch | versions | rollback`, credentials from the environment only. See its
`README.md`.

**Status (2026-07-24):** storage + import proven end-to-end against real R2 and
Meili (versioned publish, checksum-verified fetch, zero-downtime import, unified
search over the imported data). Built and pending:

- [x] R2 versioned golden-dataset layer (publish/fetch/verify/versions/rollback)
- [x] Zero-downtime Meili import (staging + atomic swap)
- [x] `searchtext` ranking + unified cross-type search (validated)
- [x] Per-type counts in the manifest (for the player)
- [ ] **Full-scale source** — ListenBrainz popularity + MB canonical dump
      (replaces the per-artist API prototype `build-sample.mjs`)
- [ ] Slim `musicmeta` to Meili-only serving + a `/stats` endpoint (counts)
- [ ] Player: unified search UI + "X songs · Y albums · Z artists indexed"
- [ ] Nightly GitHub Action (build+publish) + Railway import job
