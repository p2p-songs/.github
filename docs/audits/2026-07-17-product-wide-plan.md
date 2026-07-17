# Product-wide Plan Audit — 2026-07-17

- **Audit ID:** A-003
- **Status:** OPEN — 1 high; no implementation sign-off
- **Supersedes:** A-001 and A-002 for current plan sign-off
- **Audited commits:** `.github@0fd8abe`, `player@3fc591e`,
  `addon-sdk@3629382`, `addons@37b02be`, `backend@16e378d`
- **Last updated:** 2026-07-17
- **Open issue:** [addon-sdk#1](https://github.com/p2p-songs/addon-sdk/issues/1)
- **Registry:** [`README.md`](./README.md)

## Scope

Adversarial review of the current committed project plan across `.github`,
`player`, `addon-sdk`, `addons`, and the optional `backend`, using the
contract's product-wide principle: audit the promise, not the parts. All repos
still contain planning/scaffolding only.

This report was revised after the product owner clarified the intended v1
scope. The audit treats those declarations as product decisions rather than
inventing requirements outside them.

## Declared scope decisions

- Addon installation by user-supplied manifest URL is intentional and required;
  an in-product addon directory/discovery flow is not a v1 requirement.
- Offline concurrent sync/conflict sophistication is out of scope for now.
- Accessibility acceptance work is deferred beyond the current phase.
- Concurrent playback on multiple devices is out of scope.
- **Basic encryption at rest remains in scope.** Advanced envelope encryption,
  KMS design, rotation automation, and disaster-recovery key ceremonies are
  later-stage hardening, not pre-implementation blockers.
- The audit should not penalize deliberately deferred features or optimize for
  hypothetical scale before a concrete requirement exists.

## Overall verdict

**DOES NOT PASS YET — one targeted protocol correction required.** The overall
architecture is sound and proportionate: web-native TypeScript, a headless
core/UI boundary, stable-ID queue entries, stamped async work,
scheduler-controlled stream commands, memory-only resolved media, official
YouTube embed, and a self-contained Torrentio-shaped debrid addon are coherent
choices. The documented legal posture also holds at plan level. One confirmed
protocol defect remains: the synthetic album-track ID collides across media in
multi-disc releases and does not clearly distinguish MusicBrainz track identity
from recording identity. No architectural rewrite is warranted.

## Six-lens coverage

- **Technical soundness:** passes except for the track-ID defect below.
- **Legal validity:** passes against current invariants. No player/backend audio
  storage, pooled debrid credential, default raw YouTube extraction, or bundled
  debrid addon is proposed. This is invariant checking, not legal advice.
- **Implementation quality:** passes at the plan level. Previously audited queue,
  async-race, stream-command, failure-bound, and bearer-media concerns are
  explicitly addressed.
- **System design and operations:** passes for the declared phase. Advanced
  offline conflict handling and encryption key lifecycle are deferred; basic
  TLS, RLS, and encryption-at-rest requirements remain.
- **UI/UX and accessibility:** passes against declared v1 scope. Manifest-based
  installation is intentional; accessibility is deferred and should be audited
  when it enters scope.
- **End-to-end user value and delight:** passes at plan level for the intended
  manifest-aware audience. Playback continuity, recovery, persistence, and
  optional resume are coherently designed; runtime delight remains to be
  verified once a vertical slice exists.

## Finding

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
  releases fixtures to protocol schema tests.
- **Verdict:** CONFIRMED
- **External verification:** MusicBrainz documents tracks as release/medium-
  scoped entities with their own MBIDs, and mediums as positioned within a
  release: https://musicbrainz.org/doc/Track and
  https://musicbrainz.org/doc/Medium

## Withdrawn findings after scope clarification

The following are not findings under the declared scope and must not be filed
as issues from this pass:

- manifest URL installation / lack of an addon directory;
- advanced UI/accessibility acceptance criteria;
- offline multi-device conflict semantics;
- self-hosted backend provisioning polish;
- advanced encryption key lifecycle beyond basic encryption at rest;
- concurrent multi-device playback ownership.

These may be audited later only if the product owner brings them into scope or
the implementation contradicts an invariant that remains in force.

## Counts

- Critical: 0
- High: 1
- Medium: 0
- Low: 0

## Sign-off condition

Correct the track identity scheme and its protocol fixtures. Once that single
finding is reconciled coherently across the plan, checklist, SDK contract, and
player/addon consumers, this plan is ready for implementation under the
declared scope.
