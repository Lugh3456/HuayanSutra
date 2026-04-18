# AI Context — Avatamsaka Sutra (Huayan Sutra) Mini Site

**Project:** Avatamsaka Sutra (大方广佛华严经) interactive reading site
**Status:** Complete — All 39 chapters done
**Last updated:** 2026-04-18
**Part of:** DharmaGate portal (https://lugh3456.github.io/DharmaGate/)
**Live URL:** https://lugh3456.github.io/HuayanSutra/
**GitHub repo:** HuayanSutra

---

## Purpose

This site makes the Avatamsaka Sutra accessible for CK's personal study. Each chapter has:
- Chinese sutra text with TTS playback
- Chinese explanations (always visible)
- English explanations (toggled, hidden by default)
- Bilingual summary

---

## Design system

All styling comes from `css/style.css`, copied verbatim from LotusSutra (which in turn came from DiamondSutra). Key tokens:

```css
--red: #b83232
--saffron: #c8820a
--ink: #1c0f0f
--bg: #f7f0e6
--card-bg: #fdf8f2
```

Fonts: Noto Serif SC (headings, body Chinese), Noto Sans SC (UI labels), loaded from Google Fonts.

---

## Page structure (all section pages)

Every `sectionN.html` follows this exact pattern:

```
portal-bar → header (h1 + nav) → main:
  full-text section (Chinese text + TTS button)
  line-explanation section (4 × explanation-card)
  summary section (Chinese + English toggle)
footer
```

JavaScript: `toggleDetail()` for show/hide English, `speak()` for TTS.

---

## Navigation pattern

- Section 1: no ← prev link (only → Contents and next)
- Sections 2–38: ← prev | Contents | next →
- Section 39 (final chapter): no → next link (end of sutra)
- As each batch of 6 is added, update the previous last chapter's header nav to add the → next link

---

## Chapter list (80-roll Śikṣānanda version, 39 chapters)

| File | Chapter | Title | Status |
|------|---------|-------|--------|
| section1.html | Ch 1 | 世主妙严品 | ✅ Done |
| section2.html | Ch 2 | 如来现相品 | ✅ Done |
| section3.html | Ch 3 | 普贤三昧品 | ✅ Done |
| section4.html | Ch 4 | 世界成就品 | ✅ Done |
| section5.html | Ch 5 | 华藏世界品 | ✅ Done |
| section6.html | Ch 6 | 毗卢遮那品 | ✅ Done |
| section7.html | Ch 7 | 如来名号品 | ✅ Done |
| section8.html | Ch 8 | 四圣谛品 | ✅ Done |
| section9.html | Ch 9 | 光明觉品 | ✅ Done |
| section10.html | Ch 10 | 菩萨问明品 | ✅ Done |
| section11.html | Ch 11 | 净行品 | ✅ Done |
| section12.html | Ch 12 | 贤首品 | ✅ Done |
| section13.html | Ch 13 | 升须弥山顶品 | ✅ Done |
| section14.html | Ch 14 | 须弥顶上偈赞品 | ✅ Done |
| section15.html | Ch 15 | 十住品 | ✅ Done |
| section16.html | Ch 16 | 梵行品 | ✅ Done |
| section17.html | Ch 17 | 初发心功德品 | ✅ Done |
| section18.html | Ch 18 | 明法品 | ✅ Done |
| section19.html | Ch 19 | 升夜摩天宫品 | ✅ Done |
| section20.html | Ch 20 | 夜摩宫中偈赞品 | ✅ Done |
| section21.html | Ch 21 | 十行品 | ✅ Done |
| section22.html | Ch 22 | 十无尽藏品 | ✅ Done |
| section23.html | Ch 23 | 升兜率天宫品 | ✅ Done |
| section24.html | Ch 24 | 兜率宫中偈赞品 | ✅ Done |
| section25.html | Ch 25 | 十回向品 | ✅ Done |
| section26.html | Ch 26 | 十地品 | ✅ Done |
| section27.html | Ch 27 | 十定品 | ✅ Done |
| section28.html | Ch 28 | 十通品 | ✅ Done |
| section29.html | Ch 29 | 十忍品 | ✅ Done |
| section30.html | Ch 30 | 阿僧祇品 | ✅ Done |
| section31.html | Ch 31 | 如来寿量品 | ✅ Done |
| section32.html | Ch 32 | 诸菩萨住处品 | ✅ Done |
| section33.html | Ch 33 | 佛不思议法品 | ✅ Done |
| section34.html | Ch 34 | 如来十身相海品 | ✅ Done |
| section35.html | Ch 35 | 如来随好光明功德品 | ✅ Done |
| section36.html | Ch 36 | 普贤行品 | ✅ Done |
| section37.html | Ch 37 | 如来出现品 | ✅ Done |
| section38.html | Ch 38 | 离世间品 | ✅ Done |
| section39.html | Ch 39 | 入法界品 | ✅ Done |

---

## Things NOT to change without asking CK

- CSS design tokens (matches LotusSutra / DiamondSutra exactly)
- English toggle pattern (hidden by default, JS-driven)
- TTS voice preference (Ting-Ting, zh-CN, rate 0.9)
- Portal bar link (always points to DharmaGate)
- Footer copyright line

---

## Workflow for adding each batch of 6 chapters

1. Write section7.html through section12.html (or next batch) with full content
2. Update the previous last chapter's header nav to add → next link
3. Update index.html: change the next 6 cards from `<div class="chapter-card coming">` to `<a href="sectionN.html" class="chapter-card">` (remove coming-label)
4. Update this ai-context.md: mark new chapters ✅ Done, shift ⏳ Next to the next batch
5. After CK reviews locally: push to GitHub

---

## Common tasks

**Add a new batch of chapters:** Follow the workflow above.

**Fix a navigation link:** Each section has ← prev, 目录 Contents, next → links in `<nav class="section-nav">`. First and last built chapters have partial nav.

**Deploy:** Push to GitHub. Pages auto-deploys from the repo root.
