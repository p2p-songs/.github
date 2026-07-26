# Uncached-download and catalog audit

- **Audit ID:** A-013
- **Status:** RECONCILED — the 1 medium addressed 2026-07-26; re-audit to confirm
- **Supersedes:** A-012 for current implementation sign-off
- **Audited commits:** `.github` `0645341898ef8db05343c54b304e4131c11a4cfc`; `player` `29545ac2e86d078716302451327541a8c7fb9b8b`; `addon-sdk` `abc0a1c91473f408d812f8ce9cae924e02e0ea23`; `addons` `a781579ea67d97ab56cda848110c58cb4a7c967f`; `backend` `682adc7ed6b5db10d37db9b8a344b65b663e17f9`
- **Last updated:** 2026-07-26

## Scope and verdict

This pass audited the product changes since A-012 across all six required
lenses, with emphasis on the new curated-catalog pipeline, default metadata
addon, unified search and album-context handoff, Bitbop candidate relevance,
uncached-download protocol and polling, Real-Debrid account mutation, and the
first-launch/search/play journey. **One documentation/UI correction is
required.** Whole-album selection is an intentional Real-Debrid constraint:
selected files become usable only after the selected set completes, and keeping
the album together makes later tracks immediately available. The implementation
and README describe that choice coherently. The configure page, however, still
promises that the generated install is cached-only, while the config it
generates omits `downloadUncached` and schema parsing turns downloading on. A
user following the supported install flow therefore receives different account
behavior from the behavior disclosed immediately before installation. No other
finding survived the checklist's evidence, present-scope, and severity gates.

## Findings

### [MEDIUM] Configure page still describes the retired cached-only behavior

- **Category:** end-to-end-user-value
- **Repo / file / line:** `addons/packages/bitbop/src/configure-page.ts:104-109,206-210`; `addons/packages/bitbop/src/config.ts:62-74`; `addons/packages/bitbop/src/resolve.ts:185-197`
- **Reference:** Adversarial Review Contract §2 UI/UX and end-to-end lenses (credential explanations and destructive surprise); not covered by an existing invariant — flagging the stale consent copy as new
- **Finding:** The only supported configure UI says Bitbop “only returns torrents your debrid account has already cached” and that uncached content would not be downloaded. Its generated config contains no `downloadUncached` field, however, so `bitbopConfigSchema` supplies the intentional default `true`. When no cached stream is found, `maybeStartDownload()` starts the best candidate and deliberately selects all audio files in the album torrent. Whole-album selection itself is not the defect: Real-Debrid requires the selected set to complete, and downloading the album makes its later tracks fast. The defect is that the installation/consent surface still describes the superseded cached-only design.
- **Why it matters:** A user can accept an explicit cached-only explanation, install the generated URL, and press one song expecting no new download. Bitbop instead adds the album to their Real-Debrid account. The behavior is defensible and beneficial, but the stale explanation creates avoidable surprise around a credentialed, state-changing action.
- **Suggested fix:** Replace the cached-only paragraph with concise, accurate copy explaining that an uncached play downloads the album on the user's Real-Debrid account and makes later tracks quick. If `downloadUncached` remains configurable, expose that choice or state the default plainly. Add a configure-page regression test so the displayed explanation cannot drift from the generated config again.
- **Verdict:** CONFIRMED
- **Resolution (2026-07-26):** The Options section's stale cached-only paragraph is replaced. It now renders a real **Download uncached tracks** checkbox — checked by default to match `bitbopConfigSchema`'s `downloadUncached: true` — and its state is written into the generated config (`downloadUncached` now rides on the base64url payload alongside `debrid`/`indexers`/`maxResults`). Revisiting `/configure` on a configured install prefills the box from the stored value. The copy states plainly that an uncached play adds the album to the user's own Real-Debrid account, downloads it with visible progress, and makes the rest of the album fast; unchecking keeps the account strictly cached-only. New `test/configure-page.test.ts` (3 tests) asserts the retired sentence is gone, the disclosure mentions the download/whole-album behavior, and — the drift guard the finding asked for — the checkbox's default-checked state is derived from and compared against `bitbopConfigSchema.parse(...).downloadUncached`, so the displayed default cannot diverge from what parsing applies. Files: `configure-page.ts` (copy, checkbox, config assembly, `readPrior`). Bitbop 200 → 205 tests, all green.

## Six-lens disposition

- **Technical soundness:** one cross-layer configuration defect. The new `resolving` protocol shape, response validation, provider fan-out, identity-stamped polling, bounded wait, candidate relevance gate, Real-Debrid reuse-by-hash, and A-012 numeric IP policy otherwise passed inspection and executable tests.
- **Legal validity:** no finding. Bitbop remains one self-contained addon, uses only the requesting user's credential and indexers, stores no audio on project infrastructure, and returns resolved HTTPS links. Whole-album selection occurs inside the user's own debrid account and does not change that posture.
- **Overall implementation quality:** the component suites, typechecks, and production builds pass. The missing test is at the composition seam: schema defaults and the generated configure payload are each locally valid, while together they reverse the UI's promise.
- **System design and operations:** no finding. Whole-album download is accepted as a coherent Real-Debrid design constraint and optimizes subsequent tracks. The curated catalog has a rebuildable golden dataset, verified import and index swap; search no longer consumes live MusicBrainz's shared rate budget. Provider failures, poll cadence, and account reuse are bounded in code. The backend remains scaffolding-only.
- **UI/UX and accessibility:** the configure page's native labeled controls and credential warning are usable, but its account-behavior explanation is materially false and the new option is absent. The player reports download progress honestly only after the hidden action has begun.
- **End-to-end user value and delight:** one copy correction is required for first install → search → press play: the first uncached play correctly prepares the album and accelerates later tracks, but the configure page describes the old cached-only behavior. Unified search, album-context propagation, downloading progress, and ready-stream handoff otherwise compose and are covered by tests.

## Verification

- Direct composition probe by source inspection: the configure page builds a config with only `debrid`, `indexers`, and `maxResults`; parsing that object applies `downloadUncached: true`; the no-stream branch calls `startDownload()` without a file picker, whose Real-Debrid adapter defaults to every audio file.
- `player`: 233/233 tests passed; typecheck and Vite production build passed.
- `addon-sdk`: 47 protocol tests and 39 SDK tests passed; typechecks and builds passed.
- `addons`: 27 MusicBrainz, 21 catalog-builder, 35 musicmeta, 25 stream-legal, and 200 Bitbop tests passed; typechecks and builds passed.
- Loopback-listener suites were rerun with local binding permission after the restricted sandbox returned `listen EPERM`; they passed unchanged. No live indexer, Meilisearch, or Real-Debrid account was used, so provider-specific download behavior beyond the directly implemented request sequence remains outside this pass.
