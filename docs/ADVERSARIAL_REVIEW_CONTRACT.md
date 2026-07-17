# Adversarial Review Contract

This is the operating agreement between two roles working on p2p-songs:

- **Implementer** — the default working session/agent. Writes code, writes
  docs, makes design calls within `IMPLEMENTATION_PLAN.md`, responds to
  findings.
- **Auditor** — a separate session/agent, invoked on demand ("come audit
  the plan/implementation"), with **no access to the implementer's
  conversation history**. It knows only what's committed to these four
  repos. Its job is to find problems, not to agree with the implementer.

If you are the auditor: you did not write any of this, you have no stake
in it being good, and your usefulness is proportional to how much you
disagree when disagreement is warranted. Don't rubber-stamp.

---

## 1. What to read, in order

1. [`docs/IMPLEMENTATION_PLAN.md`](./IMPLEMENTATION_PLAN.md) — the
   architecture and *why*. Every design decision that looks odd probably
   has a reason written down here; check before flagging it as a mistake.
2. [`docs/REVIEW_CHECKLIST.md`](./REVIEW_CHECKLIST.md) — the mechanical
   *what to check*, cross-referenced to plan sections. Treat this as your
   primary worklist for technical-soundness and legal-validity checks.
3. Each repo's `CLAUDE.md` (`player/`, `addon-sdk/`, `addons/`) — scope and
   repo-specific invariants.
4. Actual repo state: code, commit history, open issues/PRs. This is what
   you're auditing — not the plan's aspirations, the plan's aspirations
   *compared to* what's actually there.

## 2. Audit scope — three lenses, every audit

Every audit pass should explicitly cover all three, even briefly:

1. **Technical soundness** — does the implementation actually match the
   architecture in `IMPLEMENTATION_PLAN.md`? Protocol conformance (§8),
   the Torrentio-not-AIOStreams shape of `stream-debrid` (§2), correctness
   bugs, security issues (credential handling, injection, SSRF via
   user-supplied indexer URLs, etc.), missed edge cases.
2. **Legal validity** — does it actually hold the invariants in
   `REVIEW_CHECKLIST.md` §2-§5 (no content storage, per-request debrid
   credentials, `stream-ytmusic` defaults to official embed not raw
   extraction, `stream-legal` stays a closed set of CC sources)? This is
   not "give legal advice" — it's "check the code against the specific,
   already-agreed invariants," which is a factual/technical check, not a
   legal opinion.
3. **Overall implementation quality** — the normal code-review lens:
   correctness, reuse, simplification, efficiency, test coverage. If this
   environment's `code-review` skill is available to you, its category
   vocabulary (`correctness`, `simplification`, `efficiency`,
   `test-coverage`) is a fine model to reuse here.

## 3. Ground rules for the auditor

- **Verify, don't assume.** Every finding must be checked against actual
  repo state (read the file, run the check) before being reported — not
  inferred from what the plan says *should* be true.
- **Cite your source.** Every finding references either a `REVIEW_CHECKLIST.md`
  item, a specific `IMPLEMENTATION_PLAN.md` section, or, if neither
  applies, says explicitly "not covered by an existing invariant — flagging
  as new" so the implementer can tell agreed-invariant violations apart
  from the auditor's own judgment calls.
- **No invented requirements.** Don't fail the project against a
  preference that isn't in the plan or checklist unless you flag it
  explicitly as your own suggestion, separate from compliance findings.
- **Report-only. No fixing.** The auditor may read anything but must not
  modify code in `player`, `addon-sdk`, or `addons`, and must not push
  commits to any repo other than its own audit report / issues (see §4).
  Finding and fixing are different sessions on purpose.
- **Mark confidence.** Every finding gets `CONFIRMED` (you verified it
  directly against repo state) or `PLAUSIBLE` (suspected, but you weren't
  able to fully verify — e.g. behavior that depends on a live API you
  can't call).

## 4. How findings get delivered back to the implementer

Two artifacts per audit, both required:

1. **Audit report** — a new file at
   `.github/docs/audits/YYYY-MM-DD-<short-scope>.md` (e.g.
   `2026-08-01-stream-debrid.md`), committed and pushed to the `.github`
   repo. Contents: scope of this pass, one-paragraph overall verdict, then
   every finding using the template in §5. Update
   `docs/REVIEW_CHECKLIST.md`'s "Current status" section to reflect what
   was actually audited and when.
2. **GitHub Issues** — for every `critical` or `high` severity finding,
   file one issue in the specific repo the finding is about (not `.github`,
   unless the finding is about the plan/docs themselves). Title:
   `[audit] <short summary>`. Labels: `audit-finding` plus one of
   `sev:critical` / `sev:high` / `sev:medium` / `sev:low`. Body: the full
   finding (§5 template) plus a link to the audit report section it came
   from. `medium`/`low` findings can live in the report only, to avoid
   issue-tracker noise — use judgment, but err toward filing an issue if
   you'd be annoyed to see it silently dropped.

## 5. Finding template

```markdown
### [SEV] Short title
- **Category:** technical-soundness | legal-validity | implementation-quality
- **Repo / file / line:** …
- **Reference:** Plan §X, or Checklist item #Y, or "no existing invariant — new"
- **Finding:** what's wrong, stated plainly.
- **Why it matters:** concrete failure scenario — what breaks, for whom, under what input/condition.
- **Suggested fix:** optional, if obvious.
- **Verdict:** CONFIRMED | PLAUSIBLE
```

## 6. Severity rubric

- **critical** — legal-invariant violation (e.g. `stream-debrid` caching
  audio content, using a shared/pooled debrid account) or an exploitable
  security issue (credential leakage, injection, SSRF).
- **high** — a core architectural invariant is broken (e.g. `stream-debrid`
  grows an AIOStreams-style plugin interface again, `stream-ytmusic`
  defaults to raw extraction) — not yet actively harmful, but the design
  has silently drifted from what was agreed.
- **medium** — correctness bugs, protocol non-conformance, missing
  handling at a real boundary.
- **low** — code quality, simplification, style, non-blocking.

## 7. What the implementer does with findings

- Check `gh issue list --label audit-finding --state open` across the org
  repos at the start of a work session, before starting new feature work,
  so audit findings don't silently age out.
- When fixing: reference the issue in the commit (`Fixes #N` /
  `Addresses #N`) and let it auto-close, or close manually with a note.
- When disagreeing: comment on the issue explaining why, and either close
  as "won't fix" with reasoning or leave it open for the next audit pass
  to weigh in on. Don't silently ignore a finding — a closed-with-reasoning
  issue and an open-and-ignored one look identical to the next auditor
  unless you leave the comment.

## 8. Cadence

On-demand is the primary trigger — ask for an audit anytime. It's also
worth running one at the end of each phase in `IMPLEMENTATION_PLAN.md` §10,
before starting the next phase, since that's when architectural drift is
cheapest to catch.
