# Addon SDK Implementation Audit

- **Audit ID:** A-005
- **Status:** CHANGES REQUIRED — 2 critical, 3 medium
- **Supersedes:** A-004 for current implementation sign-off; confirms A-004 reconciliation
- **Audited commits:** `.github` `5beae91569aac1d85a331bdbf2524a9c290dfe6b`; `addon-sdk` `edfb8654a8505027a6eb2e64171683485b70822e`; `player` `af63ee23746ff6d601aa420439bed5540845c6dd`; `addons` `459cb5bb5017ae69d29f7e5c5b948e62e51b8493`; `backend` `682adc7ed6b5db10d37db9b8a344b65b663e17f9`
- **Last updated:** 2026-07-19

## Scope

This pass re-audits the four A-004 protocol fixes and audits Phase 2's
`@p2p-songs/addon-sdk`: builder, handler types, framework-neutral router,
`node:http` adapter, configuration round-trip, default configuration page, and
their composition with the standalone wire specification. The later player,
reference-addon, and backend implementations remain scaffolding, so their
future-phase promises were checked for compatibility but not failed as missing.
The audit included source/package inspection, runtime adversarial probes, all
68 tests, typecheck, build, repository state/history, and all six contract
lenses.

## Overall verdict

A-004 is confirmed reconciled: HTTPS-only resource URLs, type/identity
discrimination, playlist identity, the standalone v0.1 spec, and the explicit
bonus-disc fixture are present and tested. Phase 2 is compact and its happy
path works, but it is not safe to sign off as the credential-carrying addon
boundary. Configured manifest responses are explicitly public-cacheable even
though their request URL contains the user's secret, and arbitrary handler
exception messages are returned to callers verbatim. The router also accepts
wire-invalid content types, ignores `configurationRequired`, and lets malformed
percent encoding reject the router promise instead of producing a controlled
client error. These failures are at the shared SDK boundary and would be
inherited by every reference addon.

## Findings

### [CRITICAL] Secret-bearing configured manifest URLs are marked public-cacheable

- **Category:** technical-soundness
- **Repo / file / line:** `addon-sdk/packages/sdk/src/router.ts:73-84,193-200`
- **Reference:** Checklist §7 configured-addon URL secret handling; Plan §3/§7
- **Finding:** The router correctly recognizes a configured URL's leading path segment, but always serves `/<encoded-config>/manifest.json` with `Cache-Control: public, max-age=3600`. The encoded segment contains the user's debrid credential. Runtime/source inspection confirms the configured and unconfigured manifest paths receive the same public caching policy. Resource responses can likewise opt into `public` caching while their request URL contains the same segment.
- **Why it matters:** A browser intermediary, reverse proxy, or CDN is explicitly invited to retain a request keyed by a URL containing a bearer credential. This contradicts the project's rule that configured addon URLs be excluded from HTTP/SW caching and unnecessarily spreads a debrid key into cache metadata and operational surfaces. Anyone with cache/log visibility may recover and reuse the credential.
- **Suggested fix:** Treat any request with a decoded config prefix as secret-bearing and emit `Cache-Control: no-store, private` (and an appropriate `Vary`/deployment warning) for the manifest and resource responses. Never apply shared/public caching to a configured path; add tests for every configured route.
- **Verdict:** CONFIRMED

### [CRITICAL] Handler exception details can disclose addon credentials

- **Category:** technical-soundness
- **Repo / file / line:** `addon-sdk/packages/sdk/src/router.ts:103-135`; `addon-sdk/packages/sdk/src/serve.ts:35-39`
- **Reference:** Checklist §7 (secrets are never logged or exposed); Plan §7
- **Finding:** The router serializes `err.message` from every handler exception into the remote JSON response's `detail`; the HTTP adapter does the same for router-level failures. A runtime probe using config `{key:"SECRET-123"}` and a representative provider error returned `{"detail":"provider rejected key SECRET-123"}` to the caller.
- **Why it matters:** Debrid/indexer clients commonly include request URLs, headers, or provider response text in thrown errors. An unauthenticated caller can exercise addon routes and receive those internal messages, turning a routine provider failure into disclosure of the configured credential or internal service details. The SDK is the trust boundary that should make the safe behavior the default.
- **Suggested fix:** Return a stable opaque error body to clients; make diagnostic reporting opt-in through a redacting logger that never receives raw configured URLs or credentials. Add regression tests with a sentinel secret in both handler and router-level exceptions.
- **Verdict:** CONFIRMED

### [MEDIUM] Stream and lyrics routes accept wire-invalid content types

- **Category:** technical-soundness
- **Repo / file / line:** `addon-sdk/packages/sdk/src/router.ts:94-130`; `addon-sdk/packages/sdk/src/types.ts:15-35`
- **Reference:** Protocol §5; Checklist §6 content types and recording-keyed stream/lyrics
- **Finding:** The router validates the recording and album-context IDs but passes the route's `type` through as an arbitrary string. A runtime probe of `/stream/artist/mbid:recording:….json` returned 200 and invoked the stream handler with `type: "artist"`, although the wire spec requires `type` to be `track`. Handler types reinforce the gap by declaring `type: string`.
- **Why it matters:** Independently implemented clients/addons can disagree about route semantics while both appear SDK-valid. Cache keys, handler branching, and future authorization/routing can then operate on a contradictory content type instead of rejecting a malformed request at the boundary.
- **Suggested fix:** Validate all route content types; require literal `track` for stream/lyrics and a protocol `ContentType` for catalog/meta. Prefer resource-specific request schemas/types and add negative route tests.
- **Verdict:** CONFIRMED

### [MEDIUM] `configurationRequired` does not prevent unconfigured resource calls

- **Category:** legal-validity
- **Repo / file / line:** `addon-sdk/packages/sdk/src/router.ts:73-130`; `addon-sdk/packages/protocol/src/manifest.ts:31-39`; `addon-sdk/docs/PROTOCOL.md` §7
- **Reference:** Protocol §7; Checklist §3 per-request debrid credentials
- **Finding:** `behaviorHints.configurationRequired` is descriptive only. The router calls resource handlers with `config: undefined` even when it is true; malformed config segments are silently stripped and treated the same way. The existing test explicitly demonstrates the unconfigured handler invocation. A runtime probe with a non-base64 config prefix still returned 200.
- **Why it matters:** The highest-scrutiny consumer, `stream-debrid`, must use credentials from that request and must not fall back to operator credentials. The shared SDK currently makes an absent or corrupted credential an addon-specific concern, allowing a handler mistake/fallback to cross the legal invariant instead of failing closed at the transport boundary.
- **Suggested fix:** When `configurationRequired` is true, reject or return the resource's empty container before invoking a handler unless a valid config object was decoded. Treat a malformed explicit config prefix as 400 rather than silently downgrading it to unconfigured; test both cases.
- **Verdict:** CONFIRMED

### [MEDIUM] Malformed percent-encoding escapes the router's error boundary

- **Category:** implementation-quality
- **Repo / file / line:** `addon-sdk/packages/sdk/src/router.ts:70-103,149-170`; `addon-sdk/packages/sdk/src/extra.ts:9-15`; `addon-sdk/packages/sdk/src/serve.ts:35-39`
- **Reference:** Protocol §6 error behavior
- **Finding:** `decodeURIComponent` is called by `parseResourcePath` and `parseExtra` before the handler `try/catch`. A runtime request containing `/%E0%A4%A.json` rejected the framework-neutral router promise with `URIError: URI malformed` instead of returning a `4xx`. Under `serveHTTP` this becomes a 500 and again exposes the exception detail.
- **Why it matters:** Any unauthenticated client can turn malformed input into an internal-error path. Non-Node adapters expecting the router's documented `Promise<RouterResponse>` behavior must add their own catch logic, and monitoring is polluted with attacker-controlled 500s for ordinary bad requests.
- **Suggested fix:** Put all parsing inside a boundary that converts malformed URI components into a stable 400 response; add malformed ID and extra-segment tests against both `createRouter` and `serveHTTP`.
- **Verdict:** CONFIRMED

## Six-lens summary

- **Technical soundness:** Changes required: two credential-boundary failures and incomplete request validation are confirmed above. A-004's schema corrections are confirmed.
- **Legal validity:** Changes required at the shared enforcement seam: `configurationRequired` fails open. No reference addon yet exists, so no content-storage, source-license, pooled-account, YT extraction, or open-proxy violation was found in implemented code.
- **Overall implementation quality:** The code is small, typed, and green on 68 tests/typecheck/build, but malformed input escapes the router abstraction and adversarial boundary cases are absent from tests.
- **System design and operations:** The framework-neutral router plus thin Node adapter is a justified simplification. Public caching and raw diagnostic responses are unsafe defaults for deployments carrying path credentials; later deployment/migration/observability promises are not implemented yet.
- **UI/UX and accessibility:** The default configure page has a label, language, viewport, keyboard-operable controls, and escaped server-rendered values. No blocking accessibility finding in this limited page. Its invalid-JSON `alert` and raw-JSON workflow are basic, but reference addons may override it and no concrete primary product UI exists yet.
- **End-to-end user value and delight:** The SDK hello-world journey works over HTTP, but the real first-launch-to-play journey cannot run until reference addons/player land. Current error behavior would expose internals rather than give an honest actionable state; no claim of product readiness is made.

## Verification record

- `pnpm test`: 10 files, 68 tests passed (localhost permission required only for the intended live HTTP test).
- `pnpm typecheck`: passed.
- `pnpm build`: passed.
- Runtime probes confirmed: `stream/artist` returns 200; malformed config is treated as absent; a sentinel secret in a handler exception is returned verbatim; malformed percent encoding rejects `createRouter` with `URIError`.
- Source/runtime inspection confirmed configured manifests receive `Cache-Control: public, max-age=3600`.
- A-004 reconciliation confirmed: 46 protocol tests cover HTTPS scheme rejection, type↔ID identity, playlist IDs, and the explicit bonus-disc fixture; `docs/PROTOCOL.md` exists and agrees with the implemented shapes reviewed here.
- No code in `player`, `addon-sdk`, `addons`, or `backend` was modified by this audit.
- No GitHub issues were created; publication awaits explicit user approval.
