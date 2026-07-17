# Core Player Architecture Re-audit — 2026-07-17

## Scope

Second plan-level audit of the committed core-player architecture after commit
`4afaee3` in `player` and commit `a6a69c9` in `.github`. This pass deliberately
goes beyond checking whether the prior findings were edited: it pressure-tests
the protocol boundary, stream-resolution semantics, queue invariants, async
race handling, credential threat model, and failure behavior. No player runtime
exists yet, so findings concern implementation-shaping defects in the committed
design rather than observed runtime failures.

## Overall verdict

**CHANGES REQUIRED, but no architectural rewrite is justified.** The central
direction is sound: a web-native layered engine, a small reducer-style playback
FSM, a separate resolution scheduler, a headless core/UI boundary, Dexie for
durable library data, and memory-only resolved streams are all appropriate.
Returning to a global Elm runtime, Rust/WASM, or a premature package split
would make this project worse. However, the design is not implementation-ready
at two critical seams. First, its centerpiece expiry logic consumes an
`expiresAt` value the wire protocol never supplies. Second, it applies generic
query retry/SWR behavior to `/stream`, even though that request may cause
rate-limited or state-changing debrid work. Four narrower state/security issues
should also be resolved while the engine is still only a plan.

## Prior audit disposition

All four findings from `2026-07-17-core-player-plan.md` are genuinely resolved
in the committed documentation: configured addon URLs are classified as
secrets, the master plan no longer mandates Elm/Rust, resolved media is
memory-only, and gapless playback is now a measured target. None is reopened
in this report.

## Lens coverage

- **Technical soundness:** the macro-architecture passes; the protocol/scheduler
  contract, `/stream` execution policy, queue identity model, and async
  completion rules require changes.
- **Legal validity:** passes at plan level. The player remains source-neutral,
  bundles no stream addon, stores no audio bytes, handles only resolved URLs or
  official `ytId` playback, and does not speak BitTorrent.
- **Implementation quality:** module boundaries and test seams are strong. The
  plan needs explicit invariants for secret storage, stale async results,
  multi-failure termination, and queue mutation under shuffle.

## Findings

### [HIGH] Scheduler expiry logic depends on a field absent from the protocol

- **Category:** technical-soundness
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:129-134,256-267`; `.github/docs/IMPLEMENTATION_PLAN.md:268-299`
- **Reference:** Plan §8; ARCHITECTURE §4a/§5; Checklist §6; no existing invariant — new
- **Finding:** The scheduler's core correctness rule stores and checks
  `expiresAt`, but the authoritative stream object contains only `url`, `name`,
  and `behaviorHints` (or `ytId`). No expiry field or derivation policy exists.
  The player therefore cannot know when a debrid URL is "past (or near)"
  expiry without provider-specific URL parsing, which would violate addon
  neutrality.
- **Why it matters:** The centerpiece JIT guarantee cannot be implemented from
  the wire contract. A player may preload a link that expires seconds before
  use, discard a still-valid link too early, or add forbidden knowledge of
  specific debrid URL formats. Fake-resolver TTL tests would pass against a
  capability real addons never provide.
- **Suggested fix:** Add an optional protocol-level expiry signal such as
  `behaviorHints.expiresAt` (absolute UTC timestamp) or a clearly defined
  `maxAgeSeconds`, validated by the SDK. Define conservative player behavior
  when absent: memory-only use within a short local horizon, and re-resolve on
  load/auth failure. Keep provider knowledge inside the addon.
- **Verdict:** CONFIRMED

### [HIGH] Generic retry and stale-while-revalidate are unsafe for `/stream`

- **Category:** technical-soundness
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:37,89-103,247-277,281-304,565-568`; `.github/docs/IMPLEMENTATION_PLAN.md:45-80`
- **Reference:** Plan §2; ARCHITECTURE §5/§6; no existing invariant — new
- **Finding:** The architecture assigns all addon HTTP responses, explicitly
  including `/stream`, to TanStack Query's caching, retries, SWR, cancellation,
  and refetch machinery. A `stream-debrid` request is not necessarily a passive
  lookup: the plan permits it to trigger a debrid-side download and poll. Even
  cache-check/unrestrict calls are credentialed and rate-limited. Automatic
  retry, focus refetch, reconnect refetch, or stale revalidation can therefore
  repeat expensive work outside the scheduler's intent.
- **Why it matters:** Merely focusing a tab or reconnecting can consume debrid
  quota, trigger duplicate torrent jobs, race two returned bearer URLs, or
  defeat the scheduler's cancellation/rate-control guarantees. Treating
  metadata reads and stream resolution as one query policy is a category error.
- **Suggested fix:** Split the clients by semantics. Use TanStack Query with
  normal cache/SWR policy for manifest/catalog/meta/lyrics. Route `/stream`
  through a scheduler-owned resolution command with explicit in-flight
  deduplication, `retry: false` (or a narrowly classified retry policy),
  `refetchOnWindowFocus: false`, `refetchOnReconnect: false`, memory-only result
  storage, and operation IDs. If TanStack Query remains the transport wrapper,
  define a separate immutable policy factory for stream keys.
- **Verdict:** CONFIRMED

### [MEDIUM] Queue cursor and shuffle order encode two incompatible positions

- **Category:** implementation-quality
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:126-154,501-513`
- **Reference:** ARCHITECTURE §4a/§8a; no existing invariant — new
- **Finding:** `cursor` is defined as an index into canonical `items`, while
  `order` is a permutation describing playback order. The plan never defines
  whether advance increments `cursor`, advances within `order`, or maps between
  both. Its UI example incorrectly defines "Up next" as `items` after the
  cursor, which is not the shuffled playback sequence. Insert/remove/reorder
  operations can also invalidate index permutations.
- **Why it matters:** Shuffle, next/previous, queue editing, persistence, and
  autoplay can disagree about the current item or skip/duplicate tracks—the
  central object of the player becomes ambiguous before code is written.
- **Suggested fix:** Use stable queue-item IDs throughout. Keep
  `itemsById`/`canonicalOrder: QueueItemId[]`, derive or store
  `playOrder: QueueItemId[]`, and represent position as `currentItemId` plus a
  playback-order position if history semantics need it. Define mutation
  invariants and make "up next" read from `playOrder`, not canonical order.
- **Verdict:** CONFIRMED

### [MEDIUM] AbortController alone does not prevent stale resolution commits

- **Category:** technical-soundness
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:156-193,247-277,525-533`
- **Reference:** ARCHITECTURE §4b/§5; no existing invariant — new
- **Finding:** Cancellation is specified only as aborting requests after a
  skip/reorder. Aborts race with completion, and some work may already have
  completed or may not honor abort. The reducer/scheduler contract has no
  operation ID, queue-item ID check, session generation, or rule for rejecting
  late `resolved`/`failed` events.
- **Why it matters:** Rapid skips or queue replacement can let an old request
  overwrite the current item's resolution, preload the wrong URL, or move the
  playback FSM from `resolving` to `buffering` for a track no longer selected.
- **Suggested fix:** Stamp every resolve/load attempt with an immutable
  `{ sessionEpoch, queueItemId, attemptId }`. Reducer events carry the stamp and
  are ignored unless it matches current state. Abort remains an optimization;
  identity validation is the correctness mechanism. Test resolve-after-skip,
  failure-after-success, reorder-during-resolve, and double-completion races.
- **Verdict:** CONFIRMED

### [MEDIUM] A separate IndexedDB store is not a browser keychain

- **Category:** implementation-quality
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:308-345`
- **Reference:** Checklist §7; ARCHITECTURE §6a; no existing invariant — new
- **Finding:** The corrected plan properly treats configured addon URLs as
  secrets, but calls a dedicated Dexie object store "conceptually the
  keychain." Object-store separation provides organization, not a security
  boundary: any same-origin script or successful XSS can read the raw URL. The
  plan lists redaction and cache controls but no browser threat model or XSS
  defenses proportional to storing a debrid credential.
- **Why it matters:** The main realistic player-side credential theft path is
  malicious/injected same-origin code, not accidental cross-store reads. The
  current language may give implementers false confidence while third-party UI,
  theming, analytics, and PWA code share the same origin.
- **Suggested fix:** Rename it a secret-bearing store, not a keychain, and add a
  v1 threat model: strict CSP without unsafe inline/eval, Trusted Types where
  supported, no remote theme/plugin code, dependency and telemetry rules,
  redacted error boundaries, and tests proving service-worker exclusion.
  Client-side encryption without a user-held key should not be presented as a
  solution because it does not stop same-origin script.
- **Verdict:** CONFIRMED

### [MEDIUM] Failure skip-ahead has no termination or circuit-breaker rule

- **Category:** implementation-quality
- **Repo / file / line:** `player/docs/ARCHITECTURE.md:163-177,268-277`
- **Reference:** ARCHITECTURE §4b/§5; no existing invariant — new
- **Finding:** After all streams for an item fail, the machine always skips to
  the next item so playback "never halts." The plan does not define behavior
  when every item fails, when repeat-all wraps, when autoplay keeps appending
  unresolvable items, or when the same addon is globally unavailable.
- **Why it matters:** A provider outage can produce an unbounded resolve/fail/
  skip loop, hammer APIs, grow radio queues, and leave the UI apparently busy
  instead of surfacing a terminal session error.
- **Suggested fix:** Track a failure sweep per playback session/addon. Stop and
  expose an actionable terminal state after every eligible item has failed once
  (or after a bounded consecutive-failure threshold); reset the breaker on user
  action or a successful play. Apply exponential backoff to provider-wide
  failures and do not let repeat/autoplay bypass the bound.
- **Verdict:** CONFIRMED

## Recommended architecture after corrections

Keep the current four-layer design. Narrow it at the unstable seams:

1. **Metadata query plane:** TanStack Query for manifest/catalog/meta/lyrics,
   with ordinary dedupe/cache/SWR behavior.
2. **Resolution command plane:** scheduler-owned `/stream` operations with
   explicit operation IDs, strict retry/refetch policy, memory-only results,
   and protocol-provided optional expiry.
3. **Queue state:** stable item IDs plus separate canonical and playback-order
   ID sequences; never use mutable array indices as durable identity.
4. **Playback orchestration:** reducer remains pure; a small controller emits
   stamped effects and discards stale completions. No global Elm runtime or
   XState dependency is necessary unless the actual machine grows materially.
5. **Failure/security policy:** bounded failure sweeps and a documented browser
   secret threat model enforced through CSP, dependency constraints, redaction,
   and cache tests.

## Counts

- Critical: 0
- High: 2
- Medium: 4
- Low: 0

