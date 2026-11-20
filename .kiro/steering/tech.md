# Tech stack and conventions

- Runtime: Node 22, TypeScript (strict), ES modules.
- Server: Fastify. Endpoints: `POST /api/scan` (start job, returns id), `GET /api/scan/:id/events` (SSE progress), `GET /api/scan/:id` (final result).
- Jobs are held in memory with a TTL (no database). One Railway service serves API + built frontend.
- HTTP: `undici`. HTML parsing: `cheerio`. Concurrency: `p-limit`. robots.txt: `robots-parser`. IP checks: `ipaddr.js`.
- Frontend: React + Vite + Tailwind.
- Tests: vitest for unit tests, fast-check for property-based tests. Every module in `src/core` has tests.
- Pure logic (URL normalization, classification, snapshot selection, SSRF decisions) lives in `src/core` as side-effect-free functions so it can be property-tested.
- Prefer small, dependency-light code. Ask before adding a new dependency.
- Config via environment variables only; document them in `.env.example`.
