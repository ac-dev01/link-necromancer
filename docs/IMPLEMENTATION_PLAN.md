# Link Necromancer: implementation plan

This is the technical blueprint. Each of the 3 Kiro specs below lists the tasks it should end up with, so Kiro's generated task lists can be checked against it for gaps or extra scope.

---

## 1. Architecture

```
Browser (React)                         Railway: one Node service
┌──────────────────┐   POST /api/scan   ┌──────────────────────────────────────┐
│ ScanForm         │ ─────────────────▶ │ Fastify                              │
│ ProgressView     │ ◀── SSE events ─── │  ├─ routes/scan.ts                   │
│ Graveyard        │   GET /api/scan/id │  ├─ job-store.ts (in-memory, TTL)    │
│ ExportBar        │ ─────────────────▶ │  └─ serves built frontend (static)   │
└──────────────────┘                    │                                      │
                                        │ core/ (pure + guarded I/O)           │
                                        │  crawl → extract → check → classify  │
                                        │             └→ wayback (dead only)   │
                                        │  all network → safeFetch (SSRF-proof)│
                                        └──────────────────────────────────────┘
                                                 │                │
                                           target sites    archive.org Wayback API
```

**Pipeline per scan:** validate URL → fetch robots.txt → BFS crawl of same-site HTML pages → extract and dedupe links → check every link → classify it → look up Wayback snapshots for the dead ones → final report.

---

## 2. Shared types (`src/core/types.ts`)

```ts
type LinkState = 'alive' | 'redirected' | 'dead' | 'unknown';

type CheckOutcome =
  | { kind: 'response'; status: number; finalUrl: string; redirects: number }
  | { kind: 'error'; code: 'DNS' | 'REFUSED' | 'RESET' | 'TIMEOUT' | 'TLS' | 'BLOCKED' | 'TOO_MANY_REDIRECTS' | 'OTHER' };

interface LinkResult {
  url: string;                 // normalized
  state: LinkState;
  outcome: CheckOutcome;
  foundOn: string[];           // pages linking here (cap 20)
  anchorText?: string;
  snapshot?: { url: string; timestamp: string };   // dead links only
}

interface ScanReport {
  id: string; startUrl: string; startedAt: string; finishedAt?: string;
  pagesCrawled: number; linksChecked: number;
  counts: Record<LinkState, number> & { resurrected: number };
  links: LinkResult[];
  status: 'running' | 'done' | 'failed'; error?: string;
}
```

---

## 3. Core modules (`src/core/`)

| Module | Responsibility | Key details |
|---|---|---|
| `url-normalize.ts` | `normalize(href, base)` → absolute URL or `null` | Resolve relative URLs. Drop `#fragment`. Lowercase the scheme and host. Remove default ports. Keep the query string as-is. Reject `mailto:`, `tel:`, `javascript:` and `data:`. |
| `ssrf.ts` | `isAllowedAddress(ip)`, `assertSafeUrl(url)` | Use `ipaddr.js` range checks. Allow only http/https. Block private, loopback, link-local, unique-local, multicast, unspecified and metadata IPs, plus IPv4-mapped IPv6. |
| `safe-fetch.ts` | The **only** network entry point | Uses an undici `Agent` with a custom `connect.lookup`. That lookup resolves DNS, rejects blocked IPs, and **pins the checked IP** so DNS rebinding can't swap it after the check. Follows redirects manually (max 5) and re-checks each hop. 10s timeout, 2 MB body cap, and a fixed User-Agent. |
| `robots.ts` | Fetch and parse robots.txt; `isAllowed(url)` | Uses `robots-parser`. If robots.txt is missing, everything is allowed. |
| `extract.ts` | `extractLinks(html, pageUrl)` → `{url, text}[]` | Uses `cheerio` on `a[href]`. Respects `<base href>`. Dedupes. |
| `crawl.ts` | BFS of same-site pages up to `maxPages` | Same site means the same host, with `www.` ignored. Only follows `text/html` responses. Emits a `page` event for each page crawled. |
| `check.ts` | `checkLink(url)` → `CheckOutcome` | Sends HEAD first. Falls back to GET when HEAD returns 405/501/403 or fails, and aborts the body as soon as headers arrive. Each URL is checked once and the result cached. |
| `limiter.ts` | Concurrency limits | A `p-limit(4)` per host plus `p-limit(16)` overall, and a 150ms gap between requests to the same host. |
| `classify.ts` | `classify(outcome)` → `LinkState` (pure) | 2xx → alive. 3xx (resolved, ends in 2xx) → redirected. 404, 410 and DNS/REFUSED errors → dead. 401/403/429, 5xx, TIMEOUT and TLS errors → unknown. |
| `wayback.ts` | `findSnapshot(url, timestamp?)` | Calls `GET https://archive.org/wayback/available?url=…&timestamp=YYYYMMDD` and reads `archived_snapshots.closest`. Concurrency 2, retries on 429/5xx with backoff, results cached. Exports a pure helper `pickClosest()` for testing. |
| `scan.ts` | Runs the whole pipeline | Emits typed events for the SSE stream. |

---

## 4. Server (`src/server/`)

- `POST /api/scan` with body `{ url, maxPages? }` → `202 { id }`. The input is validated with a schema. Limits: max 3 scans running at once overall, 1 per IP, plus `@fastify/rate-limit` at 5 scans per hour per IP.
- `GET /api/scan/:id/events` streams SSE events: `page`, `link`, `resurrect`, `progress`, `done`, `error`. Late subscribers first get the current progress.
- `GET /api/scan/:id` returns the full `ScanReport`.
- `GET /healthz` for Railway health checks.
- `job-store.ts` keeps scans in a `Map`, deletes them 1 hour after they finish, and holds at most 50 jobs.
- `@fastify/static` serves `dist/web` and falls back to `index.html` for client-side routes.

---

## 5. Frontend (`src/web/`)

| Component | Notes |
|---|---|
| `ScanForm` | URL input, a page-limit slider (10–200) and a "Begin the séance" button. Shows validation errors. |
| `ProgressView` | Live counters (pages crawled, links checked, deaths found, resurrected) and a candle progress bar. An `aria-live` region announces progress. |
| `Graveyard` | Tabs/filters by state, search, and sort (most-linked first). Laid out as a responsive tombstone grid. |
| `Tombstone` | Shows the URL, the cause of death (status or error in plain words, e.g. "404 · Not found"), the pages that link to it, and a "⚡ Resurrect" link to the snapshot when one exists. |
| `ExportBar` | Markdown report, CSV, and a dead→archived replacement list. All generated in the browser from the report JSON. |
| `theme.css` | The design tokens from the `haunted-ui` steering doc. Fog and ghost animations are switched off when the user has `prefers-reduced-motion` set. |

---

## 6. Specs to create in Kiro (with the tasks each should end up with)

**Spec 1: `crawler-link-checker`** (the biggest one)
1. Project setup: `package.json`, TypeScript strict mode, vitest, fast-check, ESLint and Prettier
2. `url-normalize` + PBT
3. `ssrf` + PBT
4. `safe-fetch` with IP pinning and manual redirects
5. `robots`
6. `extract`
7. `limiter`
8. `check` (HEAD→GET fallback)
9. `classify` + PBT
10. `crawl` (BFS)
11. Fixture server and integration tests

**Spec 2: `wayback-resurrection`**
1. `wayback` client + `pickClosest` PBT
2. `scan` pipeline and its event emitter
3. Job store with TTL
4. Fastify routes and SSE
5. Concurrency and rate limits
6. API integration tests with Wayback mocked

**Spec 3: `graveyard-ui-report`**
1. Vite + React + Tailwind setup and theme tokens
2. ScanForm
3. SSE client hook (`useScan`)
4. ProgressView
5. Graveyard + Tombstone
6. Filters, search and sort
7. Exports
8. Empty, error and "no deaths found" states
9. Accessibility pass (keyboard, contrast, reduced motion)
10. Static serving + production build

---

## 7. Testing

**Property-based tests:**
- `normalize` is idempotent: `normalize(normalize(x))` equals `normalize(x)`.
- `normalize` never returns a URL with a fragment or a non-http(s) scheme.
- `isAllowedAddress` returns false for every IP generated inside a blocked range, including IPv4-mapped IPv6.
- `classify` gives exactly one state for every possible outcome, and every 2xx is `alive`.
- `pickClosest` never picks a snapshot further from the target date than any other candidate.

**Fixture server** (`test/fixtures/server.ts`), with routes:

| Route | Behaviour |
|---|---|
| `/ok` | 200 |
| `/missing` | 404 |
| `/gone` | 410 |
| `/moved` | 301 → `/ok` |
| `/loop` | redirects back to itself, to trigger the redirect limit |
| `/forbidden` | 403 |
| `/slow` | sleeps 15s, to trigger the timeout |
| `/no-head` | HEAD returns 405, GET returns 200 |
| `/redirect-internal` | 302 → `http://127.0.0.1` (must be BLOCKED) |

Pages also link to `http://dead.invalid` (`.invalid` never resolves, so it always gives a DNS failure) and contain relative links, a `<base>` tag and duplicate links.

**End-to-end:** scan the fixture site and assert the exact counts for each state.

**SSRF regression:** scanning `127.0.0.1`, `localhost`, `169.254.169.254`, `[::1]` and a domain that resolves to 10.x must all be rejected.

---

## 8. Kiro features and when to use them

| When | Feature | What |
|---|---|---|
| Day 1 | Steering | Already drafted. Have Kiro review and refine it (first journal entry). |
| Day 2 | Hook | When `src/core/**/*.ts` is saved, run vitest on the matching tests. |
| Day 2 | Hook | When `src/core/safe-fetch.ts` or `ssrf.ts` is edited, have Kiro audit the change against `crawler-ethics.md`. |
| Day 4 | Kiro CLI | Run one spec task, such as `classify`, from the CLI, so the write-up can compare CLI and IDE. |
| Day 7 | MCP | Playwright MCP, so Kiro can open the running UI, take screenshots and check layout and accessibility. |
| Day 8 | Hook | When a spec's tasks finish, update the README feature list. |
| Day 9 | Custom agent | An "Exorcist" accessibility-reviewer agent that audits the UI components. |
| Daily | Journal | One entry in `docs/KIRO_JOURNAL.md` plus a screenshot. |

---

## 9. Deployment (Railway)

- Scripts: `build` (Vite build plus `tsc` for the server), `start` (`node dist/server/index.js`). Port comes from `process.env.PORT`.
- Railway builds from the GitHub repo using its default builder (Nixpacks) on Node 22, with `/healthz` as the health check.
- Custom domain: `necromancer.<domain>` with a CNAME to Railway, which issues the HTTPS certificate.
- Env vars: `USER_AGENT_URL`, `MAX_PAGES=200`, `JOB_TTL_MS`, `RATE_LIMIT_PER_HOUR`. Document them in `.env.example`.
- Deploy a "hello" build on **day 1**, so deployment problems show up early.

---

## 10. Schedule with checkpoints

| Date | Work | Done when |
|---|---|---|
| **Nov 20–21** | Kiro refines steering; spec 1 requirements and design; project setup task; Railway "hello" deploy | The subdomain serves `/healthz` over HTTPS |
| **Nov 22** | url-normalize, ssrf, safe-fetch + PBT; the two hooks | All SSRF tests pass |
| **Nov 23** | robots, extract, limiter, check, classify | The fixture link checks are correct |
| **Nov 24** | crawl + integration tests; CLI task | A fixture scan gives exact counts |
| **Nov 25** | Spec 2: wayback, scan pipeline, job store | A real dead link resurrects |
| **Nov 26** | Routes, SSE, limits; deploy | A scan runs over the API in production |
| **Nov 27** | Spec 3: setup, theme, form, SSE hook, progress | Live progress shows in the browser |
| **Nov 28** | Graveyard, tombstones, filters, exports | Full flow works end-to-end in production |
| **Nov 29** | Empty and error states, accessibility pass, Playwright MCP review, mobile layout | Lighthouse accessibility ≥ 90 |
| **Nov 30** | Buffer, or soft-404 detection as a stretch goal | — |
| **Dec 1** | **Feature freeze.** Scan 5 real sites and fix issues. | No crashes on real sites |
| **Dec 2** | README, Kiro write-up, publish the blog post, social post | Bonus links ready |
| **Dec 3** | Record the video (under 3 minutes) and upload to YouTube as public | Video link works |
| **Dec 4** | Submit by about 10 am PT | Submitted |

---

## 11. Main risks

- **Wayback API is slow or rate-limited.** Concurrency 2, caching and backoff. The UI shows "seeking the archive…" per tombstone so slow lookups don't block the results.
- **Sites block HEAD requests or bots.** GET fallback, and those links are classed as "restless" (unknown), never "dead". That keeps false deaths out.
- **Kiro free-tier usage runs out.** Bring Kiro small, clear tasks and review each output before re-running anything. If usage gets tight, cut spec 3's polish tasks first and keep the core.
- **The app gets abused once it's public.** The SSRF guard, rate limits, page cap and job cap cover this.
