# 🪦 Link Necromancer

**Every website is a graveyard of dead links. Link Necromancer finds them and raises them from the dead.**

Paste a site URL. Link Necromancer crawls it, checks every link, and lays the dead ones to rest in a haunted graveyard — then resurrects each one from its closest Wayback Machine snapshot, so you can fix your site in minutes.

> Built for **Kiroween 2026** · Category: Costume Contest

## Status
🚧 In development (Nov 20 – Dec 4, 2026).

## How it works (planned)
| Link state | In the graveyard |
|---|---|
| Alive (2xx) | 🌿 Living |
| Redirected (3xx) | 👻 Possessed — shows its new host |
| Dead (404 / 410 / DNS failure) | 🪦 Tombstone |
| Blocked / timeout (403 / 429) | 🌫️ Restless spirit |
| Archived snapshot found | ⚡ Resurrected |

- Same-domain crawl, robots.txt respected, polite per-host rate limits
- Wayback Machine resurrection for dead links
- Live scan progress
- Export Markdown / CSV report and a dead → archived replacement list
- SSRF-safe: private, loopback and metadata addresses are always refused

## Tech
Node 22 · TypeScript · Fastify · React + Vite + Tailwind · vitest + fast-check · Railway

## How Kiro was used
See [`.kiro/`](.kiro) (steering, specs, hooks) and the build log in [`docs/KIRO_JOURNAL.md`](docs/KIRO_JOURNAL.md).

## Run locally
_Coming soon._

## License
[MIT](LICENSE)
