# CLAUDE.md — .github

This repo has no application code. It's the docs/coordination hub for the
p2p-songs project, which spans four repos under the `p2p-songs` GitHub org:

- **`.github`** (this repo) — org profile (`profile/README.md`) + docs
- **`player`** — player app + `music-core` state machine
- **`addon-sdk`** — SDK for building addons (music equivalent of `stremio-addon-sdk`)
- **`addons`** — reference addons, including `stream-debrid`

Start here regardless of which repo you're actually working in:

- [`docs/IMPLEMENTATION_PLAN.md`](./docs/IMPLEMENTATION_PLAN.md) — full
  architecture, tech decisions, and phased build order. Source of truth
  for *why* things are designed the way they are.
- [`docs/REVIEW_CHECKLIST.md`](./docs/REVIEW_CHECKLIST.md) — mechanically
  checkable invariants for adversarial/code review, cross-referenced to
  the plan. Source of truth for *what to check*.
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
