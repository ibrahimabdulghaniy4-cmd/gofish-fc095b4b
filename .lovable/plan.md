# Rewrite the in-game documentation as flowing text

## Goal
Turn the documentation into readable prose written for players, not for developers. It keeps explaining how every system works and how the in-game economy is calculated, but it never names internal files, database objects, code functions, or migration numbers.

## Content changes
- Rewrite the whole document in English as flowing paragraphs. Each system gets an explanatory narrative instead of bullet dumps.
- Keep tables only where they carry pure numbers: prices, level gates, rarity odds, XP milestones, hold tiers, withdrawal limits, quest targets, and the species list.
- Remove every mention of file names, code functions, SQL/database internals, migration numbers, and the tech stack list. Where a rule exists because it prevents cheating, explain it as "the game decides this on the server, your device can't influence it" — no implementation detail.
- Sections to cover, in this order: overview of the island and what players do; the two currencies and how they are earned and spent; how a cast is resolved (rarity, weight, weather, mutations, monster fish); XP and levels; weather and time of day; rods, baits and boats with their unlock ladder; the quest chain; leaderboard, chat and profiles; how gold value, holding tiers, and withdrawals work; a plain-language note on fairness and safety.
- Drop the developer-facing sections entirely: architecture table, data model reference, and the fix/migration history.

## Presentation changes
- Cleaner reading layout: wider comfortable measure, larger section spacing, clearer heading hierarchy, and a subtle numbering/label style for sections.
- Contents sidebar gets active-section highlighting as the reader scrolls, plus a scroll-to-top control.
- Tables restyled with zebra rows, sticky header, and softer borders so number blocks read easily inside the prose.
- Small polish: better mobile contents drawer behaviour, and a short intro card at the top of the reader.

## Technical notes
- Rewrite `src/content/game-systems.md` (content only, no game logic touched).
- Update `src/components/game/DocsPanel.tsx` for layout, active-heading tracking, and table styling.

## Checks
- Open the docs icon in the game, click through the contents, verify headings, tables, and scrolling on desktop and mobile widths.
