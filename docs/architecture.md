# Architecture — Avatamsaka Sutra (HuayanSutra) Mini Site

## Stack

Static HTML + CSS + vanilla JS. No framework, no build tool, no dependencies beyond Google Fonts (loaded via CDN).

## File organisation

```
HuayanSutra/
├── index.html              Entry point — 80-scroll grid
├── section1a.html          One file per scroll (80 total)
│   …
├── section39u.html
├── style.css               Single shared stylesheet
├── source/
│   └── scrolls/
│       ├── scroll_01.md    Authoritative Chinese source text (80 files)
│       │   …
│       └── scroll_80.md
└── docs/
    ├── ai-context.md
    ├── architecture.md
    ├── roadmap.md
    ├── audit-config.json   Audit configuration (pages.files list)
    └── reference.py        HUAYAN dict {1..80: scroll_text}
```

Generator script (not in the repo root — lives in the Cowork outputs folder):
```
outputs/generate_scrolls.py   Reads source/scrolls/*.md + embedded DATA → writes all 80 HTML files
```

## Design decisions

**Scroll-based pages, not chapter-based.** The 80-roll Śikṣānanda version (T0279) maps naturally to 80 pages — one per traditional scroll. Several shorter chapters are combined into a single scroll where the sutra itself groups them (e.g., Ch 3+4, Ch 7+8). This gives each page a meaningful textual unit without imposing an artificial chapter boundary.

**Generator script, not hand-written HTML.** All 80 pages are produced by `generate_scrolls.py`. The DATA dict in the script contains all 480 explanation card tuples and 80 bilingual summaries. Edits to content are made in the script and pages are regenerated — direct HTML editing is avoided. This makes bulk changes (template updates, nav fixes) a one-command operation.

**6 curated explanation cards per scroll.** Rather than annotating every line, each page has 6 carefully chosen passages that illuminate key Huayan philosophical concepts. Chinese explanations are always visible; English is toggled. This is intentionally sparse — audit `check_annotation_coverage` is set to `false`.

**CSS shared, JS inlined.** `style.css` is shared across all pages. JavaScript (`toggleDetail()`, `speak()`) is inlined per page — tiny, self-contained, and works if any page is opened in isolation.

**TTS targeting zh-CN.** The `speak()` function targets the zh-CN voice at rate 0.9. Falls back to any zh-CN voice if the preferred one is unavailable.

**Non-sequential filenames.** Files are named `sectionXy.html` (e.g., `section25a.html` through `section25j.html` for Ch 25's ten scrolls). The audit system uses a `pages.files` list rather than a numeric pattern to accommodate this.

## Hub-and-spoke portal model

HuayanSutra sits as a spoke off the DharmaGate hub:

```
DharmaGate (hub)
├── HeartSutra (spoke)
├── DiamondSutra (spoke)
├── LotusSutra (spoke)
├── HuayanSutra (spoke)     ← this project
└── [future spokes]
```

Each spoke is an independent GitHub repo deployed to its own GitHub Pages URL. They share the same design system (CSS tokens, fonts, portal bar) but have no code dependency on each other.

## CSS design tokens (shared verbatim from DiamondSutra/LotusSutra)

```css
--red: #b83232        /* accent for headers */
--saffron: #c8820a    /* chapter numbers, highlights */
--ink: #1c0f0f        /* primary text */
--bg: #f7f0e6         /* page background */
--card-bg: #fdf8f2    /* explanation card background */
```

## Audit system

Two Python scripts check the site before each push:

- `audit_content.py` — verifies `chineseText` element matches `reference.py`, checks card coverage (disabled for this project)
- `check_structure.py` — verifies required sections, elements, scripts, nav links, lang attribute, index links

Config: `docs/audit-config.json` with `pages.files` list (non-sequential filenames require this instead of the standard `pages.pattern`).
