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
- [ ] Content types are `artist`/`album`/`track`/`playlist`; resources are
      `catalog`/`meta`/`stream`/`lyrics`. — Plan §8
- [ ] Canonical IDs are `mbid:<musicbrainz-uuid>`; album tracks are
      `mbid:<release-mbid>:<track-number>`; `isrc:` is the secondary
      `idPrefix`. — Plan §8
- [ ] Stream objects from `stream-legal`/`stream-debrid` are always fully
      resolved (`url` present); a bare `infoHash`/`fileIdx` pointer with no
      `url` should not come from any reference addon. — Plan §8

## 7. Secrets hygiene (all repos)
- [ ] No debrid API keys, indexer credentials, or other secrets are ever
      committed to any repo, used as a default/fallback, or logged. Config
      lives only in the per-request `/configure`-encoded value.
- [ ] **Player-side reality (corrected):** a *configured* stream addon's
      manifest URL contains the user's debrid key, and the player necessarily
      holds it to call the addon — so the player must treat configured addon
      URLs as **secrets**: stored as credential material, never
      logged/exported/telemetered, config segment redacted in any UI display,
      and excluded from service-worker/HTTP caching. Do **not** accept (or
      write) a claim that "the player never holds the key" — that was audited
      as false. — Plan §7; ARCHITECTURE §6a

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

## Current status (update as phases land)
As of 2026-07-17, all four repos (`​.github`, `player`, `addon-sdk`,
`addons`) still contain planning/scaffolding only; no runtime implementation
exists. The core-player plan was audited on 2026-07-17 (verdict: changes
required — 2 high, 2 medium); **all four findings have since been reconciled
into the docs** (credential model corrected, master plan de-Elm'd and moved to
the real 4-repo layout, resolved-media persistence made memory-only, gapless
reframed as a measured target). See the Resolution section of
[`docs/audits/2026-07-17-core-player-plan.md`](./audits/2026-07-17-core-player-plan.md).
No open blocking findings; re-audit when Phase 4 code lands.
