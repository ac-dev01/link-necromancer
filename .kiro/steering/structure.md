# Project structure

```
src/
  core/     pure logic: url-normalize, crawl, check, classify, wayback, ssrf
  server/   Fastify app, routes, SSE, job store
  web/      React UI (graveyard, scan form, export)
test/
  unit/     vitest unit tests
  pbt/      fast-check property-based tests
  fixtures/ local static site with known alive/dead/redirect/blocked links
docs/       KIRO_JOURNAL.md, screenshots
.kiro/      steering, specs, hooks (always committed)
```

- File names: kebab-case. One main export per core module.
- Tests mirror source paths (`src/core/classify.ts` → `test/unit/classify.test.ts`, `test/pbt/classify.pbt.test.ts`).
