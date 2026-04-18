# Architecture — Avatamsaka Sutra (HuayanSutra) Mini Site

## Stack

Static HTML + CSS + vanilla JS. No framework, no build tool, no dependencies beyond Google Fonts (loaded via CDN).

## File organisation

```
HuayanSutra/
├── index.html          Entry point — 39-chapter grid
├── section1.html       One file per chapter (1–6 built; 7–39 to be added in batches)
│   …
├── section39.html
├── css/
│   └── style.css       Single shared stylesheet (copied verbatim from LotusSutra)
└── docs/
    └── *.md            Project documentation
```

## Design decisions

**One file per chapter.** Each `sectionN.html` is fully self-contained: no shared JS files, no includes, no template engine. The `toggleDetail()` and `speak()` functions are inlined in a `<script>` block at the bottom of every page. This is intentionally redundant — it means any page works in isolation, can be printed, can be opened as a standalone file, and has zero dependency on any other file except `css/style.css`.

**CSS shared, JS inlined.** Style is shared across all pages through a single stylesheet. JavaScript is intentionally duplicated per page because it is tiny (< 30 lines) and the isolation benefit outweighs the duplication cost.

**Explanation cards: Chinese visible, English toggled.** The default state assumes a Chinese-literate reader. English explanations are available but hidden, revealed by a toggle button. This keeps the reading experience clean for the primary use case.

**TTS targeting Ting-Ting.** The `speak()` function prefers the Ting-Ting zh-CN voice, which produces the most natural Classical Chinese reading on macOS/iOS. Falls back to any zh-CN voice if Ting-Ting is unavailable. Rate is set to 0.9 (slightly slower than default) for sutra reading.

**Incremental build (6 chapters per session).** Index cards for unbuilt chapters use `<div class="chapter-card coming">` (not `<a>`) with `opacity: 0.5` and `pointer-events: none`. When a batch is built, those divs are replaced with live `<a>` links.

**No next link on the current last built chapter.** The last built section's nav does not include a → next link. When the next batch is added, the previous last section's nav is updated to add it. This keeps navigation honest and prevents dead links.

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
