# Product-wide Plan Audit — 2026-07-17

## Scope

Adversarial review of the current committed project plan across `.github`,
`player`, `addon-sdk`, `addons`, and the new optional `backend`. This pass uses
the contract's product-wide principle: audit the promise, not the parts. It
traces first-run-to-play, queue/playback recovery, durable restoration,
configured-addon secrets, and cross-device resume. All repositories still
contain planning/scaffolding only, so findings are about implementation-shaping
decisions rather than runtime defects.

## Overall verdict

**DOES NOT PASS YET.** The core technical direction is good and should not be
rewritten: web-native TypeScript, a headless core/UI boundary, stable-ID queue,
stamped async work, scheduler-owned stream commands, memory-only bearer media,
official YouTube embed, and a self-contained Torrentio-shaped debrid addon are
sound choices. The legal invariants also remain intact at plan level. The plan
fails product-wide sign-off for two reasons: a new user cannot reach first
playback without already knowing an addon manifest URL, and the canonical track
ID scheme collides on multi-medium releases. The optional sync feature also
lacks deterministic conflict, server-discovery, secret-key-management, and
active-device semantics. Finally, the UI phase specifies theme machinery more
concretely than usability, accessibility, and recovery acceptance criteria.

## Six-lens coverage

- **Technical soundness:** macro-architecture passes; track identity and sync
  ordering do not.
- **Legal validity:** passes against current invariants. No player/backend audio
  storage, pooled debrid account, default raw YouTube extraction, or bundled
  debrid addon is proposed. Public multi-tenant credential custody remains
  explicitly unblessed. This is invariant checking, not a legal opinion.
- **Implementation quality:** boundaries and test seams are strong. Sync
  versioning, encryption key lifecycle, and multi-device ownership remain too
  vague to implement consistently.
- **System design and operations:** local-only operation is coherent. Optional
  self-hosted sync lacks provisioning/discovery and operational acceptance
  criteria beyond "run Docker on a rented box."
- **UI/UX and accessibility:** fails. The plan lists surfaces and themes but no
  complete onboarding, loading/error/empty/offline, keyboard, screen-reader,
  responsive, or reduced-motion acceptance criteria.
- **End-to-end user value and delight:** fails at time-to-first-success. Playback
  design is unusually thoughtful once a source is installed, but source
  installation assumes knowledge the product has not taught or supplied.

## Journey results

- **First launch → source → search → play:** blocked before source installation;
  no discovery/deep-link/onboarding mechanism exists beyond pasting a URL.
- **Album/queue playback and failure recovery:** plan-level pass; prefetch,
  stamped completion, fallback, and bounded failure sweep are coherent.
- **Reload/offline restoration:** plan-level pass; identity persists and bearer
  media does not.
- **Configured secret lifecycle:** local rules pass; synced encryption lifecycle
  remains underspecified.
- **Cross-device resume:** plausible happy path, but concurrent-device and
  backend-discovery semantics are missing.
- **Keyboard/mobile/assistive technology:** not specified, so cannot pass.

## Findings

### [HIGH] First-run playback requires out-of-band manifest knowledge

- **Category:** end-to-end-user-value
- **Repo / file / line:** `.github/docs/REVIEW_CHECKLIST.md:16-22`; `player/docs/ARCHITECTURE.md:808-817,832-834`; `.github/docs/IMPLEMENTATION_PLAN.md:95-103,423-444`
- **Reference:** Plan §3/§10; Checklist §1; no existing invariant — new
- **Finding:** Source installation is exclusively "user pastes a manifest URL."
  The plan defines no onboarding explanation, trusted directory, install deep
  link, QR flow, configure-to-install return path, or even a way for a user to
  discover the safe `stream-legal`/`stream-ytmusic` manifests. The first-run
  product is therefore an empty player that requires undocumented knowledge
  from outside the product.
- **Why it matters:** A normal user can install the PWA, see no music, and have
  no actionable route to success. Search, playback quality, themes, and sync
  are irrelevant if the primary journey ends at a blank addon manager. This is
  not mere onboarding polish; it is a hard prerequisite for the app's core job.
- **Suggested fix:** Preserve neutrality but design a user-initiated install
  journey: an empty state explaining metadata vs stream addons; HTTPS install
  links/deep links that open the player with host + permissions shown; a
  configure page that returns to the player; QR/copy flow for other devices;
  and a clearly labeled directory or documentation link for independently
  operated addons. If first-party safe addons are listed, distinguish
  "available" from "preinstalled" and require an explicit user install.
- **Verdict:** CONFIRMED

### [HIGH] Album-track IDs collide across discs and confuse track with recording

- **Category:** technical-soundness
- **Repo / file / line:** `.github/docs/IMPLEMENTATION_PLAN.md:268-299`; `.github/docs/REVIEW_CHECKLIST.md:63-75`; `addon-sdk/CLAUDE.md:16-20`
- **Reference:** Plan §8; Checklist §6
- **Finding:** The canonical album-track form is
  `mbid:<release-mbid>:<track-number>`. Track position is scoped to a MusicBrainz
  medium, so disc 1 track 1 and disc 2 track 1 collide. Track number can also be
  free text (for example `A4`). Separately, MusicBrainz distinguishes a track
  (a recording as represented on one release/medium) from a recording, and
  tracks already have their own MBIDs; the plan's generic "track MBID" does not
  say which entity it means.
- **Why it matters:** Multi-disc albums cannot be queued, persisted, deduplicated,
  streamed, or synced reliably. Two different entries can address the same
  resource URL/cache key, while the same recording across releases may be
  incorrectly treated as different or identical depending on the caller.
- **Suggested fix:** Define entity semantics explicitly. Prefer the MusicBrainz
  track MBID for release-track identity and recording MBID for recording-level
  identity. If a synthetic route is necessary, include medium identity or
  position plus track identity—never release + track number alone. Add
  multi-disc, vinyl/free-text number, bonus-disc, and same-recording-on-two-
  releases fixtures to the protocol schema tests.
- **Verdict:** CONFIRMED
- **External verification:** MusicBrainz documents tracks as release/medium-
  scoped entities with their own MBIDs, and mediums as positioned within a
  release: https://musicbrainz.org/doc/Track and
  https://musicbrainz.org/doc/Medium

### [MEDIUM] UI architecture has no usability or accessibility acceptance contract

- **Category:** ui-ux-accessibility
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:587-680,730-765,801-817`; `.github/docs/IMPLEMENTATION_PLAN.md:434-444`
- **Reference:** ARCHITECTURE §7/§8a/§10 P-5; no existing invariant — new
- **Finding:** The plan specifies theme tokens, a typed theme surface contract,
  headless hooks, and runtime theme selection, but P-5 has no acceptance
  criteria for onboarding, navigation, loading/skeleton states, empty states,
  partial addon results, credential failure, offline behavior, focus order,
  keyboard playback/queue editing, screen-reader announcements, contrast,
  reduced motion, touch targets, or responsive layout.
- **Why it matters:** The architecture can produce a technically swappable,
  visually polished UI that is confusing or inaccessible in every important
  state. Deferring all product behavior until after P-1 through P-4 also delays
  discovery of core/view-model gaps until the most expensive point to change
  them.
- **Suggested fix:** Before P-1 types freeze, define a small UX acceptance doc
  with primary flows and state matrices. Add keyboard, screen-reader, reduced-
  motion, responsive, and error-recovery criteria to P-5. Build one vertical
  walking skeleton (fake addon → search result → play/error) early; retain one
  theme and the token seam, but do not spend on structural theme substitution
  until the first UI proves the flows.
- **Verdict:** CONFIRMED

### [MEDIUM] Sync conflict ordering is not deterministic across offline devices

- **Category:** system-design
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:511-520`; `.github/docs/IMPLEMENTATION_PLAN.md:446-457`; `backend/CLAUDE.md:16-18`
- **Reference:** ARCHITECTURE §6b; Plan §10 Phase 5b; no existing invariant — new
- **Finding:** v1 uses last-writer-wins with both `updatedAt` and a "monotonic
  revision," but does not define who allocates revisions, how offline clients
  advance them, how clock skew is handled, or the deterministic tie-break when
  two devices edit the same record. Playlist item rows reduce conflict scope
  but do not define concurrent reorder representation.
- **Why it matters:** Two devices can each believe their update wins, oscillate
  on pull/push, or silently discard a newer playlist/library change. An
  `updatedAt` supplied by client clocks is not a total order; an offline client
  cannot allocate a globally monotonic revision without a server compare-and-
  swap rule.
- **Suggested fix:** Define an authoritative version protocol: server-assigned
  revision plus conditional writes (`baseRevision`/ETag), deterministic conflict
  records, tombstones, and a tie-break rule. Use stable fractional/order keys or
  an explicit move operation for playlist order. Test skewed clocks, offline
  edits, delete-vs-update, reorder-vs-add, and retry after timeout.
- **Verdict:** CONFIRMED

### [MEDIUM] Self-hosted sync has no server provisioning or discovery journey

- **Category:** end-to-end-user-value
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:468-492,522-544`; `.github/docs/IMPLEMENTATION_PLAN.md:446-457`; `backend/CLAUDE.md:3-18,36-41`
- **Reference:** ARCHITECTURE §6b; Plan §10 Phase 5b; Checklist §9
- **Finding:** The product promises login and cross-device resume, but the
  supported backend model requires the user to rent a server and deploy a
  multi-container Supabase stack. The plan does not define how the player learns
  the backend URL, validates TLS, pairs a second device, handles server moves,
  exports/restores configuration, or communicates that an ordinary hosted
  account does not exist.
- **Why it matters:** "Log in on device B" is not an executable journey without
  first provisioning and locating the same server. Users may expose an insecure
  endpoint, paste the wrong origin, or abandon sync after substantial setup.
- **Suggested fix:** Treat backend setup as a first-class flow with a deployment
  health check, explicit server URL/pinning, QR pairing, TLS requirement,
  backup/restore and migration documentation, and honest UI labeling such as
  "Connect your sync server." Consider PocketBase first if the actual target is
  one-person self-hosting; Supabase is justified only if its operational cost is
  tested and accepted.
- **Verdict:** CONFIRMED

### [MEDIUM] "Encrypted at rest" lacks an implementable key lifecycle

- **Category:** system-design
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:546-578`; `.github/docs/REVIEW_CHECKLIST.md:156-174`; `backend/CLAUDE.md:22-32`
- **Reference:** Checklist §9; ARCHITECTURE §6b
- **Finding:** The plan requires the configured-URL column to be encrypted at
  rest and says the server holds the key, but does not define envelope/column
  encryption, where the master key lives, rotation, backup/restore behavior,
  disaster recovery, or whether logs/WAL/read replicas contain plaintext.
  Supabase/Postgres selection does not itself specify these controls.
- **Why it matters:** An implementation can honestly claim encrypted disks while
  database dumps, logical backups, logs, or a copied environment file expose
  every synced debrid credential. Conversely, losing the only key can make all
  synced addon configs unrecoverable.
- **Suggested fix:** Specify the threat model and key lifecycle before schema
  work: application/envelope encryption boundary, master-key storage outside
  the database, per-record/user data keys if warranted, rotation/versioning,
  redaction, backup encryption, restore drills, and deletion semantics. State
  what attacks at-rest encryption does and does not cover.
- **Verdict:** CONFIRMED

### [MEDIUM] Concurrent playback has no active-device ownership rule

- **Category:** system-design
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:501-540`
- **Reference:** ARCHITECTURE §6b; no existing invariant — new
- **Finding:** Every active device checkpoints queue identity and playback
  position every few seconds using LWW, but the plan defines neither an active
  playback session/device owner nor merge rules when two devices play at once.
- **Why it matters:** A phone and desktop left open can continually overwrite
  one another. Opening a third device may offer to resume whichever heartbeat
  happened last, not the session the user intended, while play history and
  current queue oscillate.
- **Suggested fix:** Give playback checkpoints a `sessionId`, `deviceId`,
  activity timestamp, and explicit active-session selection. Preserve separate
  session history; do not merge current position as an ordinary LWW setting.
  Let resume UI identify device/title/time and ask when ambiguity exists.
- **Verdict:** CONFIRMED

### [LOW] Master-plan tech table still says four repositories

- **Category:** implementation-quality
- **Repo / file / line:** `.github/docs/IMPLEMENTATION_PLAN.md:16-25,256-264`
- **Reference:** Plan precedence rule and §7
- **Finding:** The precedence rule correctly says five repositories, but the
  tech-stack table still says four and omits `backend`.
- **Why it matters:** Minor documentation drift undermines the plan's source-of-
  truth role and can mislead setup/automation work.
- **Suggested fix:** Change the row to five repos and name `backend` as optional.
- **Verdict:** CONFIRMED

## Counts

- Critical: 0
- High: 2
- Medium: 5
- Low: 1

## Sign-off condition

Do not rewrite the core architecture. Before implementation begins, resolve the
two high findings and define the queue/protocol fixtures and first-run vertical
slice. The medium sync findings can wait until Phase 5b, but must be resolved
before backend schema or sync-adapter implementation. UI/accessibility
acceptance criteria should be written before P-1 public types and P-5 scope are
treated as fixed.

