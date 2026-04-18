# HuayanSutra — Avatamsaka Sutra Study Site

A personal study site for the Avatamsaka Sutra (大方广佛华严经), part of the DharmaGate collection.

## Live site

https://lugh3456.github.io/HuayanSutra/

## What it is

An interactive, chapter-by-chapter reading guide for the 80-roll Śikṣānanda translation of the Avatamsaka Sutra (39 chapters). Each chapter page includes:
- A key passage of Chinese sutra text with text-to-speech playback
- Line-by-line Chinese explanations (always visible)
- English explanations (hidden by default, toggled on demand)
- A bilingual summary

Built with static HTML + CSS + vanilla JS. No framework, no build step.

## Status

All 39 chapters complete (2026-04-18).

## Folder structure

```
HuayanSutra/
├── index.html          Main page — 39-chapter grid
├── section1.html       Chapter pages (1–39, all built)
│   …
├── section39.html
├── css/
│   └── style.css       Shared stylesheet (matches LotusSutra)
└── docs/               Project documentation
```

## Deployment

Push to the `HuayanSutra` GitHub repo. GitHub Pages auto-deploys from the root.

## Part of DharmaGate

This site is a spoke of the DharmaGate portal (https://lugh3456.github.io/DharmaGate/). It shares the same design system (CSS tokens, fonts, portal bar) as DiamondSutra, LotusSutra, and HeartSutra.
