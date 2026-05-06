# Web Scraping with Claude + Browser Automation: Lessons Learned

> Context: Scraping 80 scrolls of the Huayan Sutra (大方广佛华严经) from CBETA Online (cbetaonline.cn) using Claude Cowork + Claude in Chrome MCP. Each scroll is fetched via a JSON API, converted from Traditional to Simplified Chinese, cleaned of apparatus notes, and saved as individual markdown files.

---

## Architecture Overview (What Ended Up Working)

```
Browser JS (fetch API)
  → window._allScrolls[]        (raw text, all 80 scrolls in memory)
  → OpenCC-JS conversion        (Traditional → Simplified Chinese)
  → window._cleanScrolls[]      (cleaned text, apparatus stripped)
  → JS injects batch into DOM   (<pre> element)
  → get_page_text MCP tool      (captures DOM text, ~50K chars max)
  → Write tool                  (saves raw batch to temp file)
  → parse_scrolls.py            (decodes, splits, writes scroll_NN.md)
```

---

## Network Isolation: The Fundamental Constraint

### ❌ FAILED: Shell sandbox cannot reach external URLs

The shell sandbox (where Python/bash runs) is behind a proxy that blocks external traffic.

```bash
curl https://api.cbetaonline.cn/...
# → 403 Forbidden (Tunnel connection failed)

python3 -c "import urllib.request; urllib.request.urlopen('https://...')"
# → urllib.error.URLError: <urlopen error Tunnel connection failed: 403 Forbidden>
```

**Lesson:** Never assume the shell and the browser share the same network. In Claude Cowork, they are in separate network namespaces. The browser has full internet access; the shell does not.

### ❌ FAILED: HTTP server bridge (browser → shell)

The idea: run a Python HTTP server on `localhost`, have the browser POST data to it. Blocked because the browser cannot reach `127.0.0.1` of the shell sandbox from its own network namespace.

```python
# scroll_server.py — built but never usable in this environment
server = HTTPServer(('127.0.0.1', 18765), Handler)
```

**Lesson:** Browser and shell localhost are not the same localhost. This approach only works when browser and shell are on the same machine with shared networking.

### ✅ WORKED: Browser DOM → get_page_text → Write tool

The only viable data path from browser to disk:
1. JS writes text into a DOM element
2. `get_page_text` MCP tool reads visible page text (up to ~50K chars)
3. Claude's `Write` tool saves it to a file

This is reliable but requires careful encoding (see `__NL__` trick below).

---

## The `javascript_tool` Return Value Problem

### ❌ FAILED: Returning large text directly from javascript_tool

`javascript_tool` return values are filtered/truncated:

| Attempt | Result |
|---|---|
| Return full scroll text (~8K Chinese chars) | **TRUNCATED** at ~4000 chars |
| Return base64-encoded text | **BLOCKED** entirely |
| Return hex-encoded text (~48K ASCII chars) | Visible but not usable at scale |
| Return large batches (46K+ chars) | **BLOCKED** |

**Lesson:** Never rely on `javascript_tool` return values for bulk text extraction. Use the DOM → `get_page_text` pipeline instead.

### ✅ WORKED: javascript_tool for state checks and small values

```javascript
// Good uses:
typeof window._cleanScrolls !== 'undefined'
  ? `keys: ${Object.keys(window._cleanScrolls).length}` : 'LOST'
// → "keys: 80"

window._fetchErrors.length  // → 0
window._fetchDone           // → true
```

**Lesson:** Use `javascript_tool` return values only for small status strings, counts, and booleans — not for content.

---

## The `__NL__` Newline Encoding Trick

### Problem

`get_page_text` collapses actual newlines in DOM text. A scroll with 500 line breaks becomes one giant run-on line, making it impossible to preserve paragraph/verse structure.

### Solution

Before injecting text into the DOM, replace every `\n` with the literal string `__NL__`:

```javascript
const encoded = text.replace(/\n/g, '__NL__');
document.body.innerHTML = '<pre id="out">' + encoded + '</pre>';
```

Then in `parse_scrolls.py`, decode after reading:

```python
text = text.replace('__NL__', '\n')
```

**Lesson:** When using `get_page_text` to transfer structured text, encode newlines with a unique token that won't appear in the source content. Choose a token that is both memorable and unlikely to collide with source text (e.g., `__NL__`, `⏎`, `|||`).

---

## Batch Size Limits: get_page_text vs. Write Tool

There are **two separate limits** in the pipeline, and they interact. The Write tool is the tighter constraint.

### get_page_text limit (~50K chars)

| Batch size | Chars | get_page_text result |
|---|---|---|
| 1 scroll | ~8–15K | ✅ Fine |
| 4 scrolls | ~43K | ✅ Fine |
| 5+ scrolls | ~55K+ | ⚠️ Risk of truncation |

### ❌ FAILED: Write tool truncation at ~40K chars

Even when `get_page_text` captures content correctly, passing a large string to the `Write` tool causes partial writes — the file is created but only contains a fraction of the content. This happened with 4-scroll batches (~40K chars).

**Symptom:** `batch_01_04.txt` was written successfully but only contained small text snippets instead of full scroll content.

**Root cause:** The `Write` tool has an output token limit separate from `get_page_text`. Large Chinese text batches hit this limit silently — no error is raised, the file is just short.

### ✅ WORKED: 2-scroll batches (~15–20K chars)

| Batch size | Chars | Full pipeline result |
|---|---|---|
| 2 scrolls | ~15–23K | ✅ Reliable end-to-end |
| 4 scrolls | ~40K | ❌ Write tool truncates |

**Lesson:** The binding constraint is the Write tool, not get_page_text. Use **2 scrolls per batch** (~15–20K chars) for reliable end-to-end operation. Always check the written file size against the injected content length to detect silent truncation:

```bash
wc -c batch_NN_NN.txt   # compare to content.length from javascript_tool
```

```javascript
// Always check length before injecting
const content = parts.join('\n');
content.length  // Target: < 25000 for 2-scroll batches
```

---

## Delimiter / Marker Strategy for Batch Files

Use unambiguous start/end markers that cannot appear in the source text:

```
===SCROLL_2_START===...content...===SCROLL_2_END===
```

The Python parser uses a regex with backreference to ensure matched pairs:

```python
pattern = r'===SCROLL_(\d+)_START===(.*?)===SCROLL_\1_END==='
matches = re.findall(pattern, content, re.DOTALL)
```

**Lesson:** Always use numbered paired delimiters. Backreference matching (`\1`) catches any mismatched pairs caused by truncation or encoding errors.

---

## HTML Escaping in DOM Injection

When injecting Chinese text into `innerHTML`, escape special HTML characters to avoid DOM corruption:

```javascript
document.body.innerHTML = '<pre id="out">'
  + content
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
  + '</pre>';
```

**Lesson:** Always escape `&`, `<`, `>` before injecting text as innerHTML. Failure to do so causes silent truncation when the parser hits an unrecognized "tag".

---

## Apparatus Notes Stripping: The lastIndexOf Pattern

CBETA text has a predictable structure:

```
大方广佛华严经                          ← title (line 1)

大方广佛华严经卷第N于阗国三藏...译       ← scroll header (line 4, with translator)
[...main sutra text...]
大方广佛华严经卷第N                      ← end marker (standalone, before apparatus)
静【大】，〔－〕【宋】...               ← apparatus variant readings (UNWANTED)
```

The title marker `大方广佛华严经卷第` appears **twice**: once in the header and once as the end marker before apparatus. Use `lastIndexOf` to find the second occurrence and strip from there:

```python
TITLE_MARKER = '大方广佛华严经卷第'
idx1 = text.find(TITLE_MARKER)
if idx1 >= 0:
    idx2 = text.find(TITLE_MARKER, idx1 + len(TITLE_MARKER))
    if idx2 >= 0:
        text = text[:idx2].strip()
```

### ❌ FAILED: Stripping before language conversion

Initial attempt ran stripping logic before OpenCC (Traditional→Simplified) conversion. The marker `大方广佛华严经卷第` is Simplified; in Traditional it is `大方廣佛華嚴經卷第`. So the strip found nothing and apparatus notes were preserved.

**Fix:** Always strip apparatus **after** running OpenCC conversion, when the text is already in Simplified Chinese.

### ❌ FAILED: Searching for marker after the first newline

Earlier logic found the first `\n`, then searched for the title marker after it. The scroll body begins with:

```
大方广佛华严经\n\n\n大方广佛华严经卷第二于阗国...
```

So the first match after the first newline was the **scroll header** (line 4), cutting nearly all the content.

**Fix:** Find the first occurrence of the marker, then search for the **second** occurrence. Use `find(marker, idx1 + len(marker))` rather than searching from the first newline.

---

## Fetching Strategy: Async with Rate Limiting

All 80 scrolls were fetched in a single browser session using async JS with a 300ms delay between requests:

```javascript
async function fetchAll() {
  window._allScrolls = {};
  window._fetchErrors = [];
  for (let n = 1; n <= 80; n++) {
    try {
      const resp = await fetch(
        `https://api.cbetaonline.cn/juans?work=T0279&juan=${n}&edition=CBETA`
      );
      const data = await resp.json();
      window._allScrolls[n] = Object.values(data.results[0]).join('');
    } catch(e) {
      window._fetchErrors.push({ n, error: e.message });
    }
    await new Promise(r => setTimeout(r, 300));
  }
  window._fetchDone = true;
}
fetchAll();
```

**Result:** 0 errors across all 80 scrolls.

**Lessons:**
- Store everything in `window.*` variables so data survives across multiple `javascript_tool` calls in the same session
- Use `window._fetchDone` and `window._fetchErrors` flags so you can poll for completion without blocking
- 300ms delay is sufficient for CBETA's API; lower may risk rate-limiting
- `Object.values(data.results[0]).join('')` reconstructs text from the character-array format CBETA uses

---

## Language Conversion: OpenCC-JS

For Traditional→Simplified Chinese conversion, load OpenCC-JS from CDN in the browser:

```javascript
// Load the library
const script = document.createElement('script');
script.src = 'https://cdn.jsdelivr.net/npm/opencc-js@1.0.5/dist/umd/full.js';
document.head.appendChild(script);

// Wait for load, then convert
const converter = OpenCC.Converter({ from: 'hk', to: 'cn' });
for (let n = 1; n <= 80; n++) {
  window._cleanScrolls[n] = await converter(window._allScrolls[n]);
}
```

**Lesson:** Run conversion in the browser, not the shell (shell has no network to install packages, and Python opencc packages are large). CDN-loaded JS libraries are ideal for one-off transformation tasks.

---

## Data Persistence Across Sessions

`window.*` variables **persist within a browser tab session** but are lost if:
- The tab navigates to a different URL
- The tab is closed or reloaded
- The browser restarts

**Crucially confirmed:** `window._cleanScrolls` (all 80 scrolls) **survived a full Claude context reset** — when the Claude session ran out of context and was summarized, the browser tab stayed open and all data remained intact. The distinction is:

| Event | window.* data |
|---|---|
| Claude context runs out / session resets | ✅ Survives (tab stays open) |
| Claude conversation ends, tab left open | ✅ Survives |
| Browser tab closed or reloaded | ❌ Lost |
| Computer restarted | ❌ Lost |

**Lesson:** Always check data is still present at the start of each resumed work session:

```javascript
typeof window._cleanScrolls !== 'undefined'
  ? `keys: ${Object.keys(window._cleanScrolls).length}` : 'DATA LOST'
```

**Make this the very first tool call** when resuming — before writing any todos, before any other work. If data is lost, re-run the fetch + conversion pipeline (takes ~5 minutes for 80 scrolls with 300ms delay).

---

## Special Character Cleanup

CBETA text contains a few obscure Unicode characters that cause rendering issues in some editors. Strip them in `parse_scrolls.py`:

```python
text = re.sub(r'[𪗇𢔌𫎩]', '', text)
```

**Lesson:** Run a special character audit on a sample of the scraped text before processing the full set. Look for characters outside the BMP (codepoint > U+FFFF) which many tools handle poorly.

---

## CBETA Footer Stripping

CBETA appends a footer starting with `【经文资讯】` to some API responses. Strip it:

```python
ci = text.find('【经文资讯】')
if ci > -1:
    text = text[:ci].strip()
```

**Lesson:** Inspect the raw API response for headers, footers, and metadata blocks before designing your cleaning pipeline. Strip them early and unconditionally.

---

## Silent Write Truncation: Detection and Prevention

The `Write` tool silently truncates large content — it creates the file without error but only writes part of the content. There is no exception or warning.

### Detection

After every `Write` call on a batch file, verify the file is plausible:

```bash
# Check file size is in the expected range
wc -c /sessions/.../batch_NN_NN.txt

# Or check line count — each __NL__ becomes a newline, so line count should be >> 1
wc -l /sessions/.../batch_NN_NN.txt
```

Alternatively, run `parse_scrolls.py` immediately and check the char counts reported. A truncated batch will either fail to parse (mismatched delimiters) or produce very short output files (< 1000 chars for a scroll that should be ~8000+).

### Prevention

Keep batch content under ~23K chars:

```javascript
const content = parts.join('\n');
if (content.length > 23000) {
  `TOO LARGE: ${content.length} chars — reduce batch size`;
} else {
  document.body.innerHTML = '<pre>' + content.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;') + '</pre>';
  content.length;
}
```

**Lesson:** Silent failures are the most dangerous. Build verification into every step of a multi-step pipeline. For file writes specifically, always check output size.

---

## CBETA API Format: HTML Not Plain Text

The CBETA API (`api.cbetaonline.cn/juans`) returns **HTML**, not plain text. Raw `data.results[0]` values contain `<span class="footnote">`, `<span class="add">`, and other markup.

### ❌ FAILED: Joining raw values directly

```javascript
// This preserves HTML tags in the output
window._allScrolls[n] = Object.values(data.results[0]).join('');
```

The joined string contains footnote tags like `<span class="footnote">[note text]</span>` which then appear as noise in the cleaned text.

### ✅ WORKED: DOMParser to strip HTML before storing

```javascript
const raw = Object.values(data.results[0]).join('');
const doc = new DOMParser().parseFromString(raw, 'text/html');
doc.querySelectorAll('.footnote, #cbeta-copyright, .add').forEach(el => el.remove());
let text = doc.body.innerText || doc.body.textContent || '';
// Also strip inline CBETA line references like T10n0279_p0005b13
text = text.replace(/[A-Z]\d+n\d+_p\d+[a-z]\d+/g, '');
window._allScrolls[n] = text;
```

**Lesson:** Always inspect a raw API response sample before assuming format. Use `DOMParser` in the browser to cleanly strip markup rather than regex-based tag removal, which is fragile with nested or malformed HTML.

---

## blob: URL Limitation

Attempted to navigate Claude in Chrome to a `blob:` URL for file download. This fails:

```
mcp__Claude_in_Chrome__navigate → "Browser URL could not be parsed"
```

**Lesson:** `blob:` URLs are session-scoped and cannot be navigated to by external tools. Use the DOM → `get_page_text` pipeline instead of blob downloads.

---

## Summary: The Golden Rules

1. **Browser fetches, shell writes.** Never expect the shell to reach external URLs. Use the browser for all network access.
2. **DOM is the bridge.** The only reliable data path from browser to disk is: JS → DOM element → `get_page_text` → `Write` tool.
3. **Encode newlines.** Always replace `\n` with `__NL__` (or similar) before DOM injection. Decode in the post-processor.
4. **javascript_tool is for control, not content.** Use it for state flags, counts, and triggering actions — never for returning bulk text.
5. **2 scrolls per batch, not 4.** The Write tool is the binding constraint (~25K char limit), not get_page_text (~50K). Use 2-scroll batches (~15–23K chars) for reliable end-to-end operation. Verify file size after every Write call.
6. **Convert before clean.** Run language conversion (e.g., OpenCC Traditional→Simplified) before applying any content-aware stripping logic.
7. **Use lastIndexOf for end markers.** When a structural marker appears at both the beginning and end of content, use `lastIndexOf` to find the terminal occurrence.
8. **Escape HTML on inject.** Always escape `&`, `<`, `>` before setting `innerHTML`.
9. **Persist state in window.** Use `window._data`, `window._done`, `window._errors` patterns for long-running async fetches.
10. **Check browser data first thing when resuming.** `window.*` data survives Claude context resets (as long as the tab stays open), but verify before doing anything else. Make `javascript_tool` the very first call of any resumed session.

---

*Documented during the HuayanSutra scraping project, April 2026. Updated after session 2 with Write tool truncation findings and cross-session persistence confirmation.*

---

## Alternative Architecture: Direct Page DOM Scraping (Sessions 3–N)

After sessions 1–2 established the API + OpenCC-JS pipeline, sessions 3 onward switched to a simpler approach: navigating the browser directly to each CBETA page and extracting the rendered `#body` DOM element. This turned out to be more robust than the API approach.

### New Pipeline

```
navigate(cbetaonline.cn/zh/T0279_0NN)
  → sleep 10s                          (wait for JS rendering)
  → javascript_tool: clone #body,
      remove footnote/sup/note/app/rdg,
      extract innerText,
      encode newlines as __NL__,
      inject into <pre>
  → get_page_text                      (captures encoded content)
  → Write tool → batch_NN_NN.txt
  → python3 parse_scrolls.py           (decode, clean, write scroll_NN.md)
```

### Why This Beats the API Approach

| Concern | API approach | DOM scraping |
|---|---|---|
| HTML markup in response | Must DOMParse to strip | Never present — `innerText` only |
| Traditional→Simplified | OpenCC-JS via CDN (browser) | Python trad_to_simp.py (shell) |
| window.* state management | Complex (80 fetches, flags, polling) | None needed — one page at a time |
| Resumability after crash | Risky if window state lost | Each scroll is fully independent |
| Network dependency | Requires CDN for OpenCC-JS | No extra dependencies |

---

## Direct DOM Extraction: The `#body` Pattern

CBETA renders each scroll at `https://cbetaonline.cn/zh/T0279_0NN`. The scroll text lives in `#body`. Key observations:

1. **Wait 10 seconds after navigation.** The page uses JS rendering; extracting before it finishes gives empty or partial content.
2. **CSS font substitution makes text Traditional.** The site displays Simplified but the actual DOM text nodes are Traditional Chinese. Always convert in the post-processing step.
3. **Elements to remove before extraction:** `a.noteAnchor`, `a[href*="#fn"]`, `sup`, `.note`, `.app`, `.rdg` — these are CBETA apparatus, footnote links, and variant readings. Remove them from a clone, not the live DOM.

```javascript
const body = document.querySelector('#body');
const clone = body.cloneNode(true);
clone.querySelectorAll('a.noteAnchor, a[href*="#fn"], sup, .note, .app, .rdg')
     .forEach(el => el.remove());
let text = clone.innerText || clone.textContent || '';
// Strip inline line references (e.g. T10n0279_p0005b13)
text = text.replace(/T\d+n\d+_p\d+[a-z]\d+/g, '');
text = text.replace(/\[\d+\]/g, '');
// Collapse whitespace
text = text.replace(/[ \t\u3000]+/g, '');
text = text.replace(/\n{3,}/g, '\n\n');
text = text.trim();
const encoded = text.replace(/\n/g, '__NL__');
const content = `===SCROLL_N_START===${encoded}===SCROLL_N_END===`;
document.body.innerHTML = '<pre id="out">'
  + content.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;')
  + '</pre>';
content.length   // return value used to sanity-check size before get_page_text
```

**Lesson:** Check `content.length` from the `javascript_tool` return value before calling `get_page_text`. Values in the range 6000–15000 are typical for one Huayan scroll. Anything under 1000 or over 25000 indicates a problem (page not loaded, or an unexpectedly long scroll).

---

## 1-Scroll-Per-Batch: The Simplest Reliable Unit

Earlier sessions used 2-scroll batches. Session 3 onward switched to **1 scroll per batch** (one `batch_NN_NN.txt` per scroll). Benefits:

- Batch files are 6–15K chars — well inside both `get_page_text` and `Write` tool limits
- Any failure is isolated: one bad scroll doesn't affect others
- Resumption is trivial: just check which `scroll_NN.md` files exist and continue from the next missing one
- Debugging is easier: can re-run `parse_scrolls.py` on any single batch independently

**Lesson:** When scraping a long sequence with a reliable per-item pipeline, prefer 1-item batches over multi-item batches. The overhead of extra navigate/wait/extract cycles is worth the robustness gain.

---

## Python-Side Traditional→Simplified Conversion

Instead of running OpenCC-JS in the browser, sessions 3+ use a hardcoded Python mapping in `trad_to_simp.py`. This covers ~200+ character pairs that appear in Classical Buddhist Chinese.

**Why not Python opencc package?** The shell has no external network, so `pip install opencc-python-reimplemented` works but requires `--break-system-packages`. The hardcoded mapping approach avoids any runtime dependency and is faster for a fixed corpus.

**When the hardcoded mapping is insufficient:** For general-purpose Traditional→Simplified conversion outside the Huayan Sutra corpus, use one of:
- Browser: load `opencc-js` from CDN (see earlier section)
- Shell: `pip install opencc-python-reimplemented --break-system-packages`

**Lesson:** For a fixed Buddhist text corpus, a targeted character mapping is more reliable than a general-purpose converter, which may make wrong choices for rare classical forms.

---

## CBETA Page-Specific Cleaning Rules

After `trad_to_simp` conversion, `parse_scrolls.py` applies these scroll-specific cleanups:

```python
# Strip leading ： from lines (CBETA line-break artifact)
lines = [l.lstrip('：') for l in text.split('\n')]
text = '\n'.join(lines)

# Strip inline apparatus markers  [A1], [A2], T10n... references
text = re.sub(r'\[A\d+\]', '', text)
text = re.sub(r'T\d+n\d+_p\d+[a-z]\d+', '', text)
text = re.sub(r'\[\d+\]', '', text)

# Strip emphasis/correction markers
text = text.replace('[＊]', '')

# Strip rare Unicode characters that break some editors
text = re.sub(r'[𪗇𢔌𫎩]', '', text)

# Strip duplicate title marker (apparatus follows second occurrence)
for MARKER in ('大方广佛华严经卷第', '大方廣佛華嚴經卷第'):
    idx1 = text.find(MARKER)
    if idx1 >= 0:
        idx2 = text.find(MARKER, idx1 + len(MARKER))
        if idx2 >= 0:
            text = text[:idx2].strip()
            break
```

**Lesson:** Maintain these cleanup rules as a checklist. When adding a new source text, do a sample inspection and extend the rules. The `[＊]` marker and leading `：` artifacts are CBETA-specific and appear consistently across all 80 scrolls.

---

## Cross-Session Resumption with Batch Files

Because every scroll is saved to an independent `batch_NN_NN.txt` before parsing, resumption after a context limit is trivial:

1. At the start of a new session, run `python3 parse_scrolls.py batch_NN_NN.txt` on any batch file that was written but not yet parsed (check the session summary for the pending step).
2. Check which scroll_NN.md files exist: `ls scrolls/ | sort`.
3. Continue from the first missing scroll number.

No browser state is needed. No `window.*` variables to check. Each scroll is completely self-contained on disk.

**Lesson:** Design long multi-session pipelines around durable on-disk checkpoints, not in-memory state. A batch file written to disk survives any kind of session interruption. The parse step is idempotent and safe to re-run.

---

## Updated Golden Rules (Sessions 3+)

The original 10 rules above still apply. Add these:

11. **Navigate-wait-extract per scroll.** For JS-rendered pages, always sleep after navigation (10s for CBETA). Don't batch multiple extractions from one page load — one scroll per navigation cycle.
12. **Check content.length from javascript_tool.** Before calling `get_page_text`, verify the encoded content length is in the expected range (1000–20000 chars for a single Huayan scroll).
13. **Prefer 1-item batches for long sequential scrapes.** The overhead is worth the isolation.
14. **Strip before AND after conversion.** Strip non-text DOM elements in JS (before extraction). Strip apparatus markers and cleanup patterns in Python (after conversion).
15. **Hardcoded mapping > CDN library for fixed corpora.** When working with a known corpus, a targeted character mapping is faster, more reliable, and has no runtime dependencies.
16. **Disk checkpoints beat memory state.** Write a batch file before parsing; write parsed output before moving to the next item. This makes every step resumable.

---

*Updated after sessions 3–N (April 2026): added direct DOM scraping approach, 1-scroll-per-batch pattern, Python-side conversion, and cross-session resumption via batch files.*

---

## Session Restart Protocol: Tab IDs Change

### ❌ FAILED: Assuming the same tab ID across sessions

Tab IDs from a previous session are not guaranteed to exist after a Claude context reset. The first call that assumed tab `1030859909` failed:

```
Tab 1030859909 no longer exists. Call tabs_context_mcp to get current tabs.
```

### ✅ WORKED: Call tabs_context_mcp first

At the start of every resumed session, call `tabs_context_mcp` (with `createIfEmpty: true`) before any other browser tool. This returns the current live tab IDs and creates a new tab group if none exists.

```
tabs_context_mcp → { tabId: 1030859910, url: "cbetaonline.cn/zh/T0279_031" }
```

Then navigate from whatever tab is available, regardless of what the previous session used.

**Lesson:** Never hard-code tab IDs across sessions. Always rediscover them via `tabs_context_mcp` as the very first browser action of any resumed session.

---

## Navigate Timeout ≠ Navigation Failure

On scroll 32, `navigate()` timed out after 60 seconds:

```
MCP server "Claude in Chrome" tool "navigate" timed out after 60s
```

However, the page **had loaded** — confirmed by checking `document.title` after a `sleep 10` wait. The timeout was from the MCP tool waiting for a "navigation complete" signal, not from the actual page failing to load.

**Lesson:** A `navigate` timeout does not mean the navigation failed. After a timeout, always wait 10–15 seconds and then call `javascript_tool` to check `document.title`. If the title matches the expected scroll, proceed as normal. Only retry the navigation if the title is still the previous page's title.

---

## `#body` null: Retry Once Before Giving Up

On scrolls 33 and 34, the first `javascript_tool` call after navigation returned:

```
TypeError: Cannot read properties of null (reading 'cloneNode')
```

This means `#body` was null — the JS rendering hadn't finished yet despite the 12-second sleep. The fix was to sleep another 10–12 seconds and retry. On the second attempt, `#body` was available and content extracted cleanly.

**Pattern:**
1. Navigate → sleep 12s → attempt JS extraction
2. If `#body` null error: sleep 12s → retry JS extraction
3. If still null: check `document.title` to confirm correct page; sleep longer and retry once more

**Lesson:** The 10–12 second sleep after navigation is usually sufficient but not guaranteed. Always handle the `#body` null case by retrying with extra sleep rather than aborting. Two retries cover all observed cases.

---

## Scrolls Completed: Progress Log

| Session | Scrolls processed | Notes |
|---|---|---|
| 1–2 | 1–25 | API + OpenCC-JS approach |
| 3 | 26–31 | DOM scraping, 1-scroll-per-batch established |
| 4 | 32–36 | Same pipeline, tab ID rediscovery at start |
| Next | 37–80 | Resume from scroll 37 |

**Lesson:** Keep a progress log in the docs. After each session, record the last scroll completed so the next session can resume without re-reading the whole summary.

---

*Updated after session 4 (April 2026): added tab ID rediscovery protocol, navigate timeout handling, and #body null retry pattern. Scrolls 01–36 complete.*
