# Bitbop and browser-security audit

- **Audit ID:** A-011
- **Status:** OPEN — 1 critical and 2 medium findings
- **Supersedes:** A-010 for current implementation sign-off
- **Audited commits:** `.github` `c7fe50b45a9a97b36a512930fcfc905c0b929af8`; `player` `c20394199d970b2a4629e91ae113daada9acf64a`; `addon-sdk` `b9a55a7e7ea0a324da92f49ee00df70dd6e0c9c2`; `addons` `ccd15d4205000e9ae43977f8a30c0c8952a4aeaf`; `backend` `682adc7ed6b5db10d37db9b8a344b65b663e17f9`
- **Last updated:** 2026-07-21

## Scope and verdict

This pass audited the newly implemented Bitbop credential-bearing debrid addon,
the player browser-threat-model gate that landed with it, and the active
cross-repo invariants. **Changes are required.** Bitbop's request-supplied
Torznab URL is fetched without an outbound-network policy, allowing a caller of
a publicly reachable instance to target loopback, link-local, or private
services. Its debrid failure accounting also converts a total transient
provider outage into a cached no-match, and its configuration page offers two
paths that cannot currently produce streams. The request-owned credential
boundary, no-audio-storage rule, no-built-in-indexer rule, within-album file
selection, resolved-HTTPS output, player configured-URL redaction, HTTP
no-store behavior, strict production CSP, and resolved-media persistence rules
otherwise hold for the audited state.

## Findings

### [CRITICAL] User-controlled indexer URL enables server-side request forgery

- **Category:** technical-soundness
- **Repo / file / line:** `addons/packages/bitbop/src/config.ts:37`; `addons/packages/bitbop/src/indexers/torznab.ts:31`
- **Reference:** Adversarial Review Contract §2 technical-soundness lens (SSRF via user-supplied indexer URLs)
- **Finding:** Bitbop accepts any syntactically valid indexer URL and fetches it server-side. It imposes no HTTPS-only rule, destination-address policy, redirect validation, or DNS-rebinding protection.
- **Why it matters:** Anyone who can reach a publicly hosted Bitbop can encode a loopback, cloud-metadata, or internal-network URL in a configured manifest and make the server query that target. A runtime probe confirmed that `http://127.0.0.1:54321/admin` passes config validation and is requested as `http://127.0.0.1:54321/admin?apikey=x&t=search&q=a+b`.
- **Suggested fix:** Require HTTPS and validate every resolved destination and redirect hop. In public-host mode, deny loopback, link-local, multicast, private, and otherwise non-public address ranges, including after DNS resolution; defend against DNS rebinding. If private Jackett/Prowlarr is required for self-hosting, make that an explicit deployment mode rather than the public default.
- **Verdict:** CONFIRMED

### [MEDIUM] Total debrid outage is cached as a genuine no-match

- **Category:** system-design
- **Repo / file / line:** `addons/packages/bitbop/src/resolve.ts:62-80`
- **Reference:** Checklist §5 outage semantics; Checklist current-status Bitbop total-outage promise
- **Finding:** Only a `DebridError` marked as authentication failure sets `outage`. If every candidate fails because the debrid service is unavailable, rate-limited, or returning errors, each exception is swallowed and the resolver returns an ordinary empty success.
- **Why it matters:** The handler caches that response for 300 seconds, so users see "no source has this track" during a total provider outage instead of a retryable failure. This is both misleading and delays recovery after the provider returns.
- **Suggested fix:** Track attempted and failed debrid probes. When every otherwise viable candidate fails because of provider or transport errors, report an outage; reserve empty success for established uncached/no-file/no-match outcomes.
- **Verdict:** CONFIRMED

### [MEDIUM] Configuration UI offers two nonfunctional resolution modes

- **Category:** end-to-end-user-value
- **Repo / file / line:** `addons/packages/bitbop/src/configure-page.ts:80-101`; `addons/packages/bitbop/src/debrid/index.ts:19-29`; `addons/packages/bitbop/src/debrid/realdebrid.ts:56-60`
- **Reference:** Adversarial Review Contract §2a first-launch → install/configure → play journey
- **Finding:** The configure page offers AllDebrid, but the provider factory returns `undefined` for it and the handler responds with an empty result cached for 300 seconds. It also offers "Include uncached," but the Real-Debrid resolver rejects every torrent whose status is not already `downloaded`.
- **Why it matters:** Users can successfully generate and install a configuration that can never produce a stream, with no explanation that the selected functionality is unimplemented. This turns setup mistakes into opaque playback failures.
- **Suggested fix:** Remove or disable both choices until implemented, or reject them during configuration with explicit messaging. Do not serialize unsupported values into an apparently valid install URL.
- **Verdict:** CONFIRMED

## Six-lens disposition

- **Technical soundness:** changes required for the confirmed SSRF. Protocol validation, per-request credential flow, file selection, and resolved HTTPS output otherwise pass.
- **Legal validity:** no additional finding. Bitbop uses only the requesting user's credentials, stores no audio, ships no tracker list, and remains one self-contained addon.
- **Overall implementation quality:** changes required through missing hostile-destination and all-provider-failure coverage. Existing suites are otherwise comprehensive and green.
- **System design and operations:** changes required because a total debrid outage is misclassified and cached. Backend remains scaffolding-only and was not credited as implemented.
- **UI/UX and accessibility:** changes required because the configuration page advertises unavailable functionality. Its credential warning, keyboard-native controls, labels, CSP, and no-echo behavior otherwise pass.
- **End-to-end user value and delight:** changes required. The Real-Debrid cached happy path composes, but AllDebrid and uncached-mode setup lead to silent dead ends.

## Verification

- `player`: 194/194 tests, typecheck, and Vite production build passed. The built HTML has strict `script-src 'self'`, Trusted Types enforcement, and no inline script.
- `addon-sdk`: 46 protocol tests + 36 SDK tests, typechecks, and builds passed (loopback permission required for the live HTTP test).
- `addons`: 112/112 tests, typechecks, and builds passed.
- Runtime SSRF probe confirmed that a loopback HTTP indexer URL is accepted and passed to the Torznab transport.
- Repository inspection found no operator debrid credential, audio-byte persistence, built-in tracker list, raw configured-URL rendering, service worker, or persisted resolved-media URL.
