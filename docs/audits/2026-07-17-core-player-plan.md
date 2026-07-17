# Core Player Plan Audit — 2026-07-17

- **Audit ID:** A-001
- **Status:** RESOLVED — historical; current sign-off moved to A-002/A-003
- **Superseded by:** A-002 for core architecture; A-003 for product-wide sign-off
- **Audited commits:** `player@db1fb72`, `.github@36b692f`
- **Last updated:** 2026-07-17
- **Registry:** [`README.md`](./README.md)

## Scope

Plan-level audit of the core player, covering
`docs/IMPLEMENTATION_PLAN.md`, `docs/REVIEW_CHECKLIST.md`, and the committed
`player/docs/ARCHITECTURE.md`. The `player` repository contains planning
documents only, so this pass assesses whether the documented design is
internally coherent and implementation-ready; it does not claim that any
runtime defect already exists. Repository history and open audit issues were
also checked.

## Overall verdict

**CHANGES REQUIRED.** The web-native player direction is technically
reasonable, and the plan preserves the project's source-neutral legal posture,
but it is not safe to implement as written. The configured-addon transport
places debrid credentials inside a URL that the player is explicitly told to
persist, contradicting the claimed secrets boundary. Separately, the master
plan still directs Phase 4 implementers to build the retired Elm-style core and
therefore conflicts with the new player architecture at the exact point where
implementation work is sequenced. Two additional persistence and playback
risks need explicit design decisions before Phase 4/5 exit criteria can be
trusted.

## Lens coverage

- **Technical soundness:** failed due to contradictory core architecture,
  unresolved secret-bearing URL storage, ambiguous resolved-link persistence,
  and an unsupported gapless-playback guarantee.
- **Legal validity:** no proposed player-side audio storage, bundled stream
  addon, raw YouTube extraction, or BitTorrent handling was found. The neutral
  player posture is intact at the plan level, subject to fixing credential
  handling below.
- **Implementation quality:** the headless core/UI boundary, scoped FSM, JIT
  resolution horizon, cancellation, and fake-backend testing seams are good.
  The authoritative documents and persistence policies are not yet precise
  enough to prevent incompatible implementations.

## Findings

### [HIGH] Configured addon URLs make the player persist debrid credentials

- **Category:** technical-soundness
- **Repo / file / line:** `.github/docs/IMPLEMENTATION_PLAN.md:244,248`; `player/docs/ARCHITECTURE.md:261`; `.github/docs/REVIEW_CHECKLIST.md:95-97`
- **Reference:** Plan §7; Checklist §8, item 4; Checklist §7
- **Finding:** The plan encodes `/configure` JSON, including the debrid API key,
  into the manifest URL path. The player architecture then persists installed
  addon URLs in Dexie. At the same time, both the plan and checklist assert
  that debrid keys never live in or are handled by the player. All three claims
  cannot be true: persisting the configured manifest URL persists the key.
- **Why it matters:** A bearer credential embedded in a durable URL can be
  exposed through IndexedDB inspection/export, diagnostics, error telemetry,
  copied configuration, request logs, browser history, or an overly broad
  service-worker/cache policy. More basically, an implementer cannot satisfy
  the checklist while implementing the documented install mechanism.
- **Suggested fix:** Define one explicit credential model before Phase 2/4.
  If Stremio-style URL configuration is retained, acknowledge that the player
  handles an opaque secret-bearing URL and specify redaction, storage,
  logging, cache, export, and deletion rules. Prefer an opaque configuration
  identifier or fragment-to-local-secret exchange if the addon can support it;
  do not claim the player never contains the key when it stores the key-bearing
  URL.
- **Verdict:** CONFIRMED

### [HIGH] The authoritative Phase 4 plan still mandates the retired Elm core

- **Category:** technical-soundness
- **Repo / file / line:** `.github/docs/IMPLEMENTATION_PLAN.md:191-200,290-310,359-377`; `player/docs/ARCHITECTURE.md:9-14,358-403`; `.github/docs/REVIEW_CHECKLIST.md:78-90`
- **Reference:** Plan §6, §9, §10 Phase 4/6; Checklist §8
- **Finding:** The master plan partially acknowledges that the player
  architecture supersedes the Elm design, but its architecture diagram still
  labels a `music-core state machine`, its repo layout still specifies a
  separate Elm-style `music-core` package, Phase 4 explicitly requires
  `dispatch(msg) -> effects[]`, and Phase 6 still schedules a Rust/WASM port.
  `ARCHITECTURE.md` instead requires one Vite app, a scoped reducer FSM, and no
  Rust/WASM for web-only v1. The architecture document says it supersedes only
  player-specific rows in Plan §5/§7, leaving the conflicting §6/§9/§10 text
  nominally authoritative.
- **Why it matters:** An implementer following the phased source of truth will
  build the architecture the checklist explicitly says not to require. This is
  not harmless historical prose: it changes package layout, state ownership,
  Phase 4 exit criteria, and the Phase 6 roadmap.
- **Suggested fix:** Reconcile the master plan in place: replace the §6 player
  diagram, §9 layout, Phase 4 work/exit criteria, and Phase 6 language with the
  web-native architecture. State one unambiguous precedence rule rather than
  narrow supersession language.
- **Verdict:** CONFIRMED

### [MEDIUM] Queue restoration can persist expiring bearer stream URLs

- **Category:** implementation-quality
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:129-143,230-247,255-266`
- **Reference:** ARCHITECTURE §4a, §5, §6; no existing invariant — new
- **Finding:** `QueueItem.resolution` contains the chosen direct URL and expiry,
  queue snapshots are mirrored into Zustand, and the persistence policy says a
  reload restores the queue. The document does not say whether resolution
  state is stripped before persistence or whether TanStack Query persists
  stream responses. Direct debrid URLs are typically bearer links and are
  explicitly expected to expire.
- **Why it matters:** Persisting these URLs expands the credential/link exposure
  surface and restores stale state that must immediately be invalidated. It can
  also cause a reload to attempt an expired URL before re-resolution, defeating
  the JIT scheduler's stated guarantee.
- **Suggested fix:** Persist queue identity, ordering, cursor, and track metadata
  only. Treat all resolution results and stream-query cache entries as
  memory-only, redact them from diagnostics, and force `resolution: idle` on
  hydration.
- **Verdict:** CONFIRMED

### [MEDIUM] Dual audio elements do not by themselves prove gapless playback

- **Category:** technical-soundness
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:194-217,228-247`; `.github/docs/IMPLEMENTATION_PLAN.md:367-373`
- **Reference:** Plan §10 Phase 5; ARCHITECTURE §4c/§5; no existing invariant — new
- **Finding:** The design claims that two `HTMLAudioElement`s swapped on
  `ended` provide near-gapless playback, and the master plan requires a full
  album to play gaplessly as a Phase 5 exit criterion. The plan specifies URL
  preloading but no measurable transition budget, readiness threshold,
  compatible-format constraints, or test matrix. An `ended` event plus a
  JavaScript-triggered `play()` is not a sample-accurate scheduling contract.
- **Why it matters:** The flagship exit criterion can fail because of event-loop
  delay, browser buffering policy, codec/container padding, throttling, or
  autoplay restrictions even when the implementation follows the document.
  Calling the architecture complete without a measurable criterion risks
  shipping audible gaps on the primary path.
- **Suggested fix:** Replace the absolute claim with a measured target (for
  example, transition silence below a defined threshold on a named browser and
  codec matrix), distinguish crossfade from true gapless playback, and add an
  integration test using controlled same-origin media fixtures.
- **Verdict:** CONFIRMED

## Counts

- Critical: 0
- High: 2
- Medium: 2
- Low: 0

---

## Resolution (implementer response, 2026-07-17)

All four findings accepted and reconciled into the docs (no code exists yet,
so these are plan/spec fixes):

- **HIGH-1 (credential model) — fixed.** The false "no keys in the player"
  claim is corrected everywhere. New ARCHITECTURE §6a states plainly that a
  configured addon URL is a bearer secret the player necessarily holds, and
  specifies handling (dedicated Dexie keychain store, never
  logged/exported/telemetered, config segment redacted in UI, excluded from
  service-worker cache, deletable, local-only). URL-config is retained (it
  keeps the key on the user's own device); opaque-identifier alternative noted
  and rejected for v1 with reasoning. Propagated to Plan §3/§7, Checklist
  §7/§8, ARCHITECTURE §11, player/CLAUDE.md.
- **HIGH-2 (stale Elm/Phase-4/monorepo) — fixed.** Added a single precedence
  rule at the top of the plan (player → ARCHITECTURE.md is authoritative).
  Replaced §6 diagram label, §9 layout (now the real 4-repo map), §7
  monorepo/Rust rows, and Phase 4/5/6 (Phase 6 Rust port retired; Phase 4/5
  now defer to ARCHITECTURE.md's P-phases). Fixed dead names
  (`music-core`/`music-addon-sdk`/`player-app`).
- **MEDIUM-1 (persisted bearer URLs) — fixed.** ARCHITECTURE §6 persistence
  policy now persists identity/order/cursor/metadata only; resolution state and
  the `/stream` query cache are memory-only; hydration forces
  `resolution: idle`. §4a `QueueItem` annotated; invariant added to §11 and
  Checklist §8.
- **MEDIUM-2 (unproven gapless) — fixed.** ARCHITECTURE §4c now distinguishes
  crossfade (always achievable) from true gapless (not guaranteed by
  dual-element swap), sets a measured target (silence threshold on a
  browser×codec matrix, same-origin test fixtures), with crossfade as fallback.
  Plan Phase 5 exit criterion and Checklist §8 updated to match.

Verdict addressed at the plan/spec level; re-audit when Phase 4 code lands.
