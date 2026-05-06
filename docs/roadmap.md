# Roadmap — HuayanSutra

## Done

- [x] Project setup: folder structure, CSS, docs
- [x] New index.html with 80-scroll grid (scroll-based, not chapter-based)
- [x] Generator script (`outputs/generate_scrolls.py`) with full DATA for all 80 scrolls
- [x] All 80 scroll pages generated (section1a.html → section39u.html)
  - Scrolls 1–5: Ch 1 世主妙严品 (sections 1a–1e)
  - Scroll 6: Ch 2 如来现相品 (section2a)
  - Scroll 7: Ch 3+4 普贤三昧品 · 世界成就品 (section3a)
  - Scrolls 8–10: Ch 5 华藏世界品 (sections 5a–5c)
  - Scroll 11: Ch 6 毗卢遮那品 (section6a)
  - Scroll 12: Ch 7+8 如来名号品 · 四圣谛品 (section7a)
  - Scroll 13: Ch 9+10 光明觉品 · 菩萨问明品 (section9a)
  - Scroll 14: Ch 11 净行品 (section11a)
  - Scroll 15: Ch 12 贤首品 (section12a)
  - Scroll 16: Ch 13+14+15 升须弥山顶品 · 须弥顶上偈赞品 · 十住品 (section13a)
  - Scroll 17: Ch 16+17 梵行品 · 初发心功德品 (section16a)
  - Scroll 18: Ch 18 明法品 (section18a)
  - Scroll 19: Ch 19+20 升夜摩天宫品 · 夜摩宫中偈赞品 (section19a)
  - Scroll 20: Ch 21 十行品 (section21a)
  - Scroll 21: Ch 22 十无尽藏品 (section22a)
  - Scroll 22: Ch 23 升兜率天宫品 (section23a)
  - Scroll 23: Ch 24 兜率宫中偈赞品 (section24a)
  - Scrolls 24–33: Ch 25 十回向品 (sections 25a–25j)
  - Scrolls 34–39: Ch 26 十地品 (sections 26a–26f)
  - Scrolls 40–43: Ch 27 十定品 (sections 27a–27d)
  - Scroll 44: Ch 28+29 十通品 · 十忍品 (section28a)
  - Scroll 45: Ch 30+31+32 阿僧祇品 · 如来寿量品 · 诸菩萨住处品 (section30a)
  - Scrolls 46–47: Ch 33 佛不思议法品 (sections 33a–33b)
  - Scroll 48: Ch 34+35 如来十身相海品 · 如来随好光明功德品 (section34a)
  - Scroll 49: Ch 36 普贤行品 (section36a)
  - Scrolls 50–52: Ch 37 如来出现品 (sections 37a–37c)
  - Scrolls 53–59: Ch 38 离世间品 (sections 38a–38g)
  - Scrolls 60–80: Ch 39 入法界品 (sections 39a–39u)
- [x] Audit infrastructure: `docs/audit-config.json` + `docs/reference.py`
- [x] Audit passed: ✓ All 80 pages pass content and structure checks (2026-05-06)
- [x] Docs updated: ai-context.md, roadmap.md, architecture.md

## Next

- [ ] Deploy to GitHub Pages (push HuayanSutra repo)
- [ ] Update DharmaGate portal if needed (Avatamsaka card link)

## Future / nice-to-have

- [ ] Consider adding a favicon (华 character, matching DharmaGate style)
- [ ] Consider adding the sutra to DharmaGate's service worker cache
- [ ] Consider a chapter-level index within the site (not just the main scroll grid)
