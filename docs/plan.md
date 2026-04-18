# Plan — Avatamsaka Sutra (HuayanSutra) Mini Site

## Approach

Build the site incrementally: 6 chapters per session, full content each time. The index page shows all 39 chapters from the start; unbuilt chapters are greyed out with "即将上线" (coming soon).

## Session log

| Session | Date | Chapters |
|---------|------|----------|
| 1 | 2026-04-18 | Ch 1–6 (世主妙严品 → 毗卢遮那品) |
| 2 | 2026-04-18 | Ch 7–12 (如来名号品 → 贤首品) |
| 3 | 2026-04-18 | Ch 13–18 (升须弥山顶品 → 明法品) |
| 4 | 2026-04-18 | Ch 19–30 (升夜摩天宫品 → 阿僧祇品) |
| 5 | 2026-04-18 | Ch 31–39 (如来寿量品 → 入法界品) — all 39 chapters complete |

## Per-session checklist

When adding a new batch of 6 chapters:

1. Write `sectionN.html` to `sectionN+5.html` with full content (sutra text, 4 explanation cards, bilingual summary)
2. Update the previous last chapter's `<nav>` to add → next link
3. Update `index.html`: replace `<div class="chapter-card coming">` with `<a href="sectionN.html" class="chapter-card">` for the new chapters (remove coming-label span)
4. Update `docs/ai-context.md`: mark new chapters ✅, update "Next" pointer
5. Update `docs/roadmap.md`: mark batch as done
6. After CK reviews locally: push to GitHub

## Content approach per chapter

Each chapter page contains:
- **Chinese sutra text**: a key passage of 60–120 characters from the chapter, drawn from the 80-roll Śikṣānanda translation
- **4 explanation cards**: each covering one key line or idea from the passage — Chinese explanation (always visible) + English explanation (toggle)
- **Bilingual summary**: 100–150 characters Chinese + equivalent English paragraph

Content goal: substantive and accurate, pitched at a thoughtful general reader with some Buddhist background. Not academic, not simplified to the point of distortion.
