# CLAUDE.md — .github

This repo has no application code. It's the docs/coordination hub for the
p2p-songs project, which spans **five** repos under the `p2p-songs` GitHub org:

- **`.github`** (this repo) — org profile (`profile/README.md`) + docs
- **`player`** — web-only player: single Vite app, `src/core` engine + `src/ui` (web-native, NOT an Elm/`music-core` port — see its `docs/ARCHITECTURE.md`)
- **`addon-sdk`** — SDK for building addons (music equivalent of `stremio-addon-sdk`); owns the canonical `@p2p-songs/protocol` types
- **`addons`** — reference addons, including `stream-debrid`
- **`backend`** — optional, self-hosted accounts + sync (Supabase); the app works fully without it

Start here regardless of which repo you're actually working in:

- [`docs/IMPLEMENTATION_PLAN.md`](./docs/IMPLEMENTATION_PLAN.md) — full
  architecture, tech decisions, and phased build order. Source of truth
  for *why* things are designed the way they are.
- [`docs/REVIEW_CHECKLIST.md`](./docs/REVIEW_CHECKLIST.md) — mechanically
  checkable invariants for adversarial/code review, cross-referenced to
  the plan. Source of truth for *what to check*.
- [`docs/DEPLOYMENT.md`](./docs/DEPLOYMENT.md) — how player, Bitbop and the
  indexer may be arranged once the player is publicly hosted; the two config
  shapes (user-supplied vs operator-supplied indexer), which combinations are
  safe, and the browser constraints a public origin introduces.
- [`docs/ADVERSARIAL_REVIEW_CONTRACT.md`](./docs/ADVERSARIAL_REVIEW_CONTRACT.md) —
  the operating agreement between the implementer and an on-demand
  auditor: scope, ground rules, finding format, and how findings get
  delivered back (audit report + GitHub issues). If you've been asked to
  *audit* this project, start there, not here.
- [`docs/audits/`](./docs/audits/) — dated audit reports, once any exist.

This project is built with an implement/review split: one session
implements against the plan, a separate session audits against the
checklist without access to the implementer's conversation — so all three
docs need to stand on their own.

## Before implementation

Read [`docs/audits/README.md`](./docs/audits/README.md) before starting feature
work. Its first row is the latest audit and authoritative sign-off status;
issues alone do not communicate supersession or the overall verdict.
