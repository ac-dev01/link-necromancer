# Haunted UI guidelines

## Mood
Moonlit graveyard, elegant not cheesy. Spooky elements must carry meaning (each visual maps to a link state) — never decoration that hides information.

## Design tokens (starting point)
- Background: deep night `#0d0f14`; surface `#161a22`; mist `#9aa4b2`
- Text: bone `#e8e4d8`
- Accents: ectoplasm green `#7cf2a0` (resurrected/alive), candle amber `#f2b84b` (possessed/redirect), blood red `#e5484d` (dead), fog grey `#6b7280` (restless/unknown)
- Display font for headings (gothic serif), readable sans for data.

## Accessibility (required)
- WCAG 2.1 AA contrast for all text.
- State is never conveyed by color or icon alone — always include a text label.
- Full keyboard navigation, visible focus, semantic HTML, ARIA live region for scan progress.
- Respect `prefers-reduced-motion`: disable floating ghosts, fog and flicker animations.

## Copy voice
Playful-gothic but clear: "14 links have passed on. 11 can be resurrected." Technical details (status code, source page) always visible.
