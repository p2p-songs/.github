# Deployment topologies

How the three moving parts may be arranged once the player is publicly hosted,
which arrangements are safe, and which one must never be built.

The parts:

- **The player** — static files on some origin. Runs no addon, holds no
  credential of its own, and is the neutral component (Plan §3, §11).
- **Bitbop** — an addon process. Holds the *user's* debrid key for the duration
  of a request, and queries a Torznab indexer server-side.
- **Prowlarr/Jackett** — the indexer Bitbop queries.
- **musicmeta** — the metadata/catalog addon, and the one *pre-installed*
  default (§11). Hosted once, centrally; holds no user credential. Optionally
  fronts a **Meilisearch** search index.
- **Meilisearch** (optional) — a private search index behind `musicmeta`. Holds
  only public catalogue data (identity — never hashes or sources) and is never
  exposed to clients.

The first three are the **stream plane**; the last two are the **metadata
plane**, which has the opposite neutrality shape — see "The metadata plane"
below.

## The rule everything else follows

> **The address policy exists because the indexer URL is caller-supplied. It is
> a policy about _who chose the destination_, not about whether the destination
> is private.**

A caller-supplied URL fetched server-side is a confused deputy: an arbitrary
stranger picks an address and the server's network position is spent on it. That
is why `bitbop/src/net/` refuses non-public destinations by default.

An **operator-supplied** URL is not that. An operator pointing their own Bitbop
at their own Prowlarr on the same Docker network is doing what every application
does with its configured upstream. There is no deputy to confuse, and an
internal address is correct rather than suspicious.

Keeping these apart is what lets a public multi-user Bitbop reach a private
Prowlarr — the arrangement that would otherwise look forbidden.

## Two configuration shapes

**Shape A — bring your own indexer.** The user supplies the Prowlarr URL, its
API key, and their debrid key at `/configure`. Everything is in the config
segment of the manifest URL. *This is what is implemented today.*

**Shape B — operator-wired indexer** (the ElfHosted/Comet model). The operator
configures the indexer out of band; the user supplies only a debrid key. Fewer
credentials leave the user's hands and onboarding is one field instead of three.

Shape B is **not implemented**. It is not merely making the config field
optional — it changes which invariants apply, so it needs all of:

- Indexer config from the environment, never a caller-supplied default.
- When the operator has configured one, the caller's indexer field is **not
  accepted** — not "overridden", not "merged". Accepting both is how an operator
  instance silently becomes a caller-controlled fetcher again.
- The address policy is bypassed **only** for the operator-supplied URL. A
  caller-supplied URL is policed exactly as it is today, in the same process.
- `/configure` states which shape the instance is in, and which indexers it will
  query. A user cannot consent to a source they can't see.
- The legal invariant is unchanged and must be re-argued in this shape: Bitbop
  still ships no indexer list. Shape B means *this operator* chose an indexer for
  *their* instance — it must not become a default that ships in the repo, or
  "the user's own indexer" (Checklist §3) stops being true.

## Supported combinations

With the player publicly hosted:

| Bitbop | Prowlarr | Shape | Verdict |
|---|---|---|---|
| public | public | A | works — see the trust cost below |
| public | private/internal | B | works — the operator chose the destination |
| user's loopback | anything the user can reach | A | works — see browser constraints |
| **public** | **the user's private LAN** | **A** | **impossible, and must stay impossible** |

The last row is the one users will report as a bug. A publicly hosted Bitbop
cannot reach the Prowlarr on someone's home network — not primarily because of
our policy, but because it is on a different network. The workaround they will
find is `BITBOP_ALLOW_PRIVATE_INDEXERS=1`, which on a public instance turns it
into an SSRF proxy against the *operator's* infrastructure (audit A-011, A-012).

> **`BITBOP_ALLOW_PRIVATE_INDEXERS=1` is only ever valid on an instance whose
> only user is the person who started it.** It is not the answer to "my indexer
> is unreachable" on a shared instance; the answer there is Shape B or a
> publicly reachable indexer.

Note for tailnets: Tailscale addresses are `100.64/10`, which the policy denies
as CGNAT. That is correct, and it means a tailnet Prowlarr only works for a
single-user Bitbop with the flag set, or as an operator-supplied URL under
Shape B.

## The metadata plane: hosted musicmeta + a curated Meilisearch catalogue

Everything above is the **stream plane** (player → Bitbop → indexer), where
neutrality forbids the operator choosing sources. The **metadata plane** is
separate and has the opposite shape: `musicmeta` is public reference data
(entity-typed ids, names, posters; no hashes, no sources), so it is the one addon
a hosted player **pre-installs** (§11). It is seeded through the ordinary install
path, once, and a user can remove it.

Meilisearch here is **not an accelerator in front of live MusicBrainz** — that was
the earlier design and it filled the index with parodies and coupled every query
to MusicBrainz's rate budget. It is now the **curated catalogue `musicmeta` serves
from**, built offline and shipped in via a versioned dataset. Full design:
[`CATALOG_PIPELINE.md`](./CATALOG_PIPELINE.md). Topology:

```
  OFFLINE (GitHub Action)            R2 (object storage you own)      RUNTIME (Railway)
  ListenBrainz + MB canonical  ──►   datasets/catalog-<ts>.ndjson  ──►  import ──► Meilisearch
  bulk dump → build → publish        latest.json (sha256, counts)        (private) ▲
                                                                                    │ serves
   player A ┐                                                                       │
   player B ├────────────────── HTTP ──────────────────────────────►  musicmeta ───┘
   player C ┘
```

Properties that make this safe and operable:

- **One writer, and it is the offline pipeline — never a user, never
  request-time.** Players only ever *read* (a catalogue search). Nothing writes to
  the index on the request path; there is no external write surface to abuse.
- **MusicBrainz is not in the serving path at all.** It is an *offline* source for
  the nightly build, so user traffic never touches the ≤1 req/sec/IP budget. That
  is what makes one hosted `musicmeta` scale to many players.
- **Meilisearch is never exposed to clients.** It sits on a private network behind
  `musicmeta` (`meilisearch.railway.internal`). Wire it with `MEILI_URL` /
  `MEILI_API_KEY` on the `musicmeta` process.
- **The import must run inside the private network** (a Railway job, or `musicmeta`
  noticing `latest.json`'s sha256 changed), because Meili is private. The
  **build+publish** half runs off-box and only needs R2 credentials. **R2 is the
  public handoff** between the two halves — and the durable, provider-independent
  system of record, so Railway having no managed backups does not matter (restore =
  re-import the NDJSON anywhere).

This is still the **identity-only** half of the "shared index" idea, which is why
it is allowed where a shared **stream**-hash / availability index is not (Plan §3):
the catalogue points at *songs*, never at *copies* of them. Note the catalogue is
**curated/official-only, not exhaustive**; the player surfaces the indexed counts
("X songs · Y albums · Z artists") so users know the scope.

**Ready-to-run assets:** the `addons` repo ships the hosting side under
[`addons/deploy/`](https://github.com/p2p-songs/addons/tree/main/deploy) — a
`docker-compose.yml` (musicmeta + a private Meilisearch on one box), a Railway
two-service setup, off-box Meilisearch snapshots, and the Cloudflare edge (cache
rule on catalogue responses + rate limit + Bot Fight Mode). The pipeline itself is
the [`@p2p-songs/catalog-builder`](https://github.com/p2p-songs/addons/tree/main/packages/catalog-builder)
package. Meilisearch stays private behind `musicmeta`; only `musicmeta` faces
Cloudflare.

## Trust cost, stated to the user

A public Bitbop instance can read **both** credentials in a Shape A config: the
debrid key and the Prowlarr URL with its API key. That is a larger ask than
Stremio's debrid addons, where the operator sees one key. `/configure` must say
so at the point of entry — the honest framing is "the operator of this instance
can read what you type here; run your own if that is not acceptable."

Shape B reduces this to the debrid key alone, which is one of its main
arguments.

## Browser constraints once the player is public

- **Loopback yes, LAN no.** An HTTPS page may fetch `http://127.0.0.1:7003`
  because loopback is a potentially-trustworthy origin. It may **not** fetch
  `http://192.168.1.50:7003` — that is plain mixed content and the browser
  blocks it. No addon or player change can permit it. "Self-hosted addon"
  therefore means *on the machine running the browser*, unless the user puts a
  real certificate on the other host.
- **Private Network Access.** Chrome gates public→loopback requests behind a
  preflight that must be answered with `Access-Control-Allow-Private-Network:
  true`. The SDK router sends it, scoped to preflights that ask.
- **Send CSP as a response header, not only as `<meta>`.** `frame-ancestors`,
  `sandbox` and `report-uri` are **ignored** in a `<meta http-equiv>` policy, so
  the player's `frame-ancestors 'none'` currently does nothing. On a public
  origin that is missing clickjacking protection while the source reads as
  though it is present. Configure real headers at whatever serves the files and
  keep the meta tag as the fallback for the directives that do work there.

## What the operator must not do

- **Do not ship a recommended *stream*-addon list, and do not preinstall Bitbop
  or any other stream/source addon** in a hosted player. (Preinstalling the
  *metadata* addon, `musicmeta`, is fine and expected — see "The metadata plane"
  above; it names no source and cannot steer anyone to one. The prohibition is
  the stream plane.) The temptation arrives exactly when hosting makes onboarding
  friction visible. A curated list naming a debrid or source addon is the
  difference between hosting a player and distributing a piracy tool, and no code
  review catches it as a defect (Plan §3, Checklist §11).
- **Do not proxy addon traffic through the player's origin.** The player stays
  static files. Anything of the operator's that relays addon requests makes the
  operator a party to that traffic and forfeits the neutrality that makes public
  hosting defensible. The optional `backend` repo is sync-only and never sits in
  the addon path.
- **Do not pool debrid accounts.** Every debrid call uses that request's own
  credentials — no operator account, no shared key, no fallback (Checklist §3).
