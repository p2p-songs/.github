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
- [ ] `player`/`music-core` never bundles, default-installs, or hardcodes
      *any* specific stream addon (including `stream-debrid`). Addon
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

## 8. Core state machine (`player`)
- [ ] `music-core` stays plain TypeScript, Elm-style (`Msg -> Effects ->
      Model`, unidirectional). A Rust/WASM port is an explicit stretch
      phase (Plan §10, Phase 6) — flag it if attempted before Phases 1-5
      are otherwise done, since that ordering was a deliberate choice, not
      an oversight.

## Current status (update as phases land)
As of the last update, all four repos (`​.github`, `player`, `addon-sdk`,
`addons`) contain only scaffolding — a README and this documentation set.
No implementation code exists yet, so none of the above has been built or
violated. Nothing to audit yet beyond the plan itself. Update this section
(or note it in each PR) as phases from Plan §10 land.
