# Product: Link Necromancer

## One-liner
Find a website's dead links and resurrect them from the Wayback Machine — presented as a haunted graveyard.

## Audience
Bloggers, documentation maintainers, small-business site owners and open-source maintainers whose pages accumulate link rot.

## Core user flow
1. User enters a site URL (and optional page limit, default 50, max 200).
2. App crawls same-domain pages and checks every unique link, streaming live progress.
3. Results appear as a graveyard: each dead link is a tombstone showing where it appears, why it died, and a "Resurrect" link to the closest archived snapshot.
4. User exports a report (Markdown/CSV) and a dead → archived replacement list.

## Link states (the metaphor IS the function)
- alive → Living
- redirected → Possessed (show final URL)
- dead (404, 410, DNS/connection failure) → Tombstone
- unknown (403, 429, timeout) → Restless spirit
- dead + snapshot found → Resurrected

## Non-goals
No accounts, no database, no automatic PRs, no AI-generated replacements.

## Hackathon context
Kiroween 2026, Costume Contest. Judged on impact, Kiro usage, design, innovation and presentation. Polish and a complete end-to-end experience beat feature count.
