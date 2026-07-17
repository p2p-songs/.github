# p2p-songs

A Stremio-style system for music: a thin player, a stateless HTTP+JSON
addon protocol, and a debrid-backed stream addon modeled on
[Torrentio](https://github.com/TheBeastLT/torrentio-scraper) — discovery,
aggregation across indexers, and debrid resolution all inside one
self-contained addon. Built as a learning project to understand Stremio's
actual architecture by reimplementing its shape for audio.

## Repos

- [`player`](https://github.com/p2p-songs/player) — the player app + core state machine
- [`addon-sdk`](https://github.com/p2p-songs/addon-sdk) — SDK for building addons
- [`addons`](https://github.com/p2p-songs/addons) — reference addons (metadata, catalog, legal/YouTube/debrid streams, lyrics)

See [`docs/IMPLEMENTATION_PLAN.md`](./docs/IMPLEMENTATION_PLAN.md) for the
full architecture, tech decisions, and phased build order.
