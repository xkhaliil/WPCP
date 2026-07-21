# AP News — Performance Audit

## Implementer Report

**Audit date:** July 2026  
**Audited URL:** https://apnews.com/  
**Audience:** The developer assigned to act on a finding

---

## How to Reproduce This Audit

Every finding in this report was derived from Lighthouse CLI runs and Chrome DevTools. Reproduce any number before touching code:

**Lighthouse CLI (recommended — matches the baseline exactly):**

```bash
# Desktop run (matches lighthouse-desktop.json in this repo)
lighthouse https://apnews.com/ \
  --preset=desktop \
  --output=json \
  --output-path=./lh-desktop.json \
  --throttling-method=simulate \
  --chrome-flags="--headless"

# Mobile run — default throttling (matches lighthouse-mobile.json)
lighthouse https://apnews.com/ \
  --output=json \
  --output-path=./lh-mobile.json \
  --throttling-method=simulate \
  --chrome-flags="--headless"
```

**DevTools equivalent:**

1. Open Chrome. Open DevTools (`F12`).
2. Network tab → Throttling → "Slow 4G".
3. Performance tab → CPU throttling → 4× slowdown → click the record button → reload.
4. Lighthouse tab → select Mobile or Desktop → "Analyze page load".

> Note: Lighthouse results are noisy across runs. Expect ±10–15% variance in timing metrics. The qualitative findings (which resources block, which files are unused, what the LCP element is) are stable.

---

## Domain Coverage Index

Each domain investigated is listed here. Domains marked ✓ have findings below. Domains marked ○ were considered and found clean or not applicable.

| Domain                                      | Status                      |
| ------------------------------------------- | --------------------------- |
| Render-blocking critical path               | ✓ findings                  |
| CSS delivery and unused rules               | ✓ findings                  |
| JavaScript bundle structure and unused code | ✓ findings                  |
| Third-party script loading                  | ✓ findings                  |
| Image delivery and prioritization           | ✓ findings                  |
| Font loading strategy                       | ✓ findings                  |
| Layout shifts (CLS)                         | ✓ minor finding             |
| Network caching and warm-load efficiency    | ✓ findings                  |
| Accessibility                               | ✓ findings                  |
| Forced synchronous layout                   | ✓ findings                  |
| Service workers / offline support           | ○ not applicable            |
| Non-composited animations                   | ○ clean                     |
| HTTP/2 push / 103 Early Hints               | ○ not detected, not flagged |
| Web app manifest / PWA install              | ○ not applicable            |
| HTTPS and protocol security                 | ○ clean                     |

---

## F1 — Render-blocking CSS blocks first paint for 1,882 ms

**Mechanism**

The main stylesheet is loaded synchronously in `<head>`:

```html
<link
  rel="stylesheet"
  href="https://assets.apnews.com/resource/.../All.min.[hash].gz.css"
/>
```

The browser cannot render any content until this file has been fully downloaded and parsed. On desktop the measured blocking penalty is **1,882 ms**. This is the single largest render blocker on the page. The file is 107 KB on the wire (after Brotli compression) and 87.5% of its rules are unused at homepage load — the browser is stalled parsing rules for article pages, galleries, and quizzes while the homepage sits blank.

**Reproduce**

1. Lighthouse desktop run → "Render-blocking requests" audit (`render-blocking-insight`). The CSS file is listed as a render blocker with its penalty in milliseconds.
2. DevTools Performance tab → record a desktop page load → in the flame chart, find the pink "Parse Stylesheet" bar just after the HTML arrives. Its width is the blocking duration.
3. DevTools Coverage tab (`Ctrl+Shift+P` → "Show Coverage" → reload) → the CSS row will show ~87% red (unused).

**Fix**

Step 1 — identify above-the-fold rules. Run the DevTools Coverage recording on first load. Export the used-rules snapshot. The critical set is typically: CSS reset, body/html defaults, the header component, the hero card, and the primary grid column. On AP News this is likely under 15 KB uncompressed.

Step 2 — inline the critical set:

```html
<head>
  <style>
    /* critical above-the-fold rules inlined here */
  </style>
  <!-- full stylesheet loaded async, rendered before user scrolls -->
  <link
    rel="preload"
    href="/styles/All.min.[hash].gz.css"
    as="style"
    onload="this.onload=null;this.rel='stylesheet'"
  />
  <noscript
    ><link rel="stylesheet" href="/styles/All.min.[hash].gz.css"
  /></noscript>
</head>
```

Tooling options: [Critters](https://github.com/GoogleChromeLabs/critters) (webpack/Vite plugin, automates critical extraction), [Critical](https://github.com/addyosmani/critical) (Node CLI). Both can be wired into the existing build step.

**Verify**

- Lighthouse `render-blocking-insight` audit: the CSS file should no longer appear as a render blocker.
- FCP on desktop should drop from ~3.4 s toward ~0.5–1 s (TTFB is 40 ms; with no render-blocking CSS the browser can paint within the first network round trip).
- DevTools Coverage: the inline `<style>` block should be ~100% used; the async stylesheet will still show unused rules, but those are no longer blocking.

---

## F2 — Four render-blocking scripts delay first paint by 2,296 ms

**Mechanism**

Four external scripts are placed in `<head>` without `defer` or `async`, before any visible content. The browser must fetch, parse, and execute each before it proceeds:

| Script                                       | Origin                     | Blocking cost (desktop) |
| -------------------------------------------- | -------------------------- | ----------------------- |
| `apcdp.apnews.com/script.js`                 | AP News CDP / analytics    | 1,263 ms                |
| `live.primis.tech/live/liveView.php`         | Video ad loader            | 436 ms                  |
| `experiments.parsely.com/vip-experiments.js` | Parsely A/B                | 339 ms                  |
| `cdn.cookielaw.org/.../OtAutoBlock.js`       | OneTrust (GDPR auto-block) | 258 ms                  |

Total JS render-blocking penalty: **~2,296 ms** (desktop). These four scripts together with the CSS blocker (F1) account for roughly 4 seconds of blank-screen time before the browser can paint the server-rendered HTML that arrived in 40 ms.

**Reproduce**

Lighthouse desktop → "Render-blocking requests" audit. Each of the four scripts above appears in the items list with its individual blocking time.

In the DevTools Performance flame chart: these appear as purple "Evaluate Script" bars before the first green "First Contentful Paint" marker.

**Fix**

`OtAutoBlock.js` (OneTrust) **cannot** simply be deferred — it must block the page to suppress content before consent on applicable jurisdictions. However, it can be optimized:

- Confirm with Legal whether the current consent configuration actually requires this blocking mode in your target regions. Many implementations over-apply it.
- OneTrust provides an alternative non-blocking integration mode for regions where prior blocking is not required.

For the remaining three:

```html
<!-- Before: synchronous, blocking -->
<script src="https://apcdp.apnews.com/script.js"></script>
<script src="https://live.primis.tech/live/liveView.php"></script>
<script src="https://experiments.parsely.com/vip-experiments.js"></script>

<!-- After: deferred, non-blocking -->
<script src="https://apcdp.apnews.com/script.js" defer></script>
<script src="https://experiments.parsely.com/vip-experiments.js" defer></script>
<!-- Primis video ad: move to body bottom or lazy-load on IntersectionObserver -->
```

`defer` scripts execute after HTML parsing completes but before `DOMContentLoaded`. For analytics this is correct. For Parsely experiments (A/B testing) verify with the Parsely team whether deferral causes a flicker — if the experiment changes layout it may need to remain synchronous and be optimized differently (reduce script size, use edge-config for variant assignment).

**Verify**

- Lighthouse `render-blocking-insight`: the deferred scripts should no longer appear.
- Combined with F1, FCP should reach sub-1 s on desktop.
- Watch for console errors on first load confirming deferred scripts still initialize correctly.

---

## F3 — LCP element not prioritized; competes with all other images

**Mechanism**

The LCP element is the first article's hero photograph. Lighthouse identifies it as a `<img>` loaded via `dims.apnews.com` (the DIMS image CDN). This image has no `fetchpriority` attribute and no `<link rel="preload">` in `<head>`. The browser's preload scanner encounters it at the same priority as all 66 other images on the page. On desktop, LCP is **17.2 s**; on mobile, **54.3 s**.

A secondary issue: on some homepage states the top story "image" is a video element (primis.tech video player) rather than a photograph. A video player must download its JS library, initialize, and buffer before it can display a frame — orders of magnitude more expensive than a static image in the LCP slot.

**Reproduce**

1. Lighthouse → "Largest Contentful Paint element" audit → identifies the element in the DOM.
2. DevTools Elements panel: inspect the LCP `<img>`. Confirm it has no `fetchpriority` attribute and no corresponding `<link rel="preload">` in `<head>`.
3. DevTools Network tab → filter by "Img" → sort by Start Time → observe that the hero image does not start earlier than other images.

**Fix**

On the server-side template that renders the first article card, add `fetchpriority="high"` to the hero image:

```html
<!-- First article hero image only -->
<img
  src="https://dims.apnews.com/dims4/default/.../format/webp/quality/90/?url=..."
  fetchpriority="high"
  loading="eager"
  alt="[story description]"
  width="1440"
  height="960"
/>
```

Do **not** add `loading="lazy"` to this element — `lazy` and `fetchpriority="high"` conflict. All other below-fold images should have `loading="lazy"`.

For the optional `<link rel="preload">` in `<head>` (further improvement): this requires the server to know which image URL will appear as the LCP element before the HTML is rendered, which is practical with SSR since the homepage content is server-assembled:

```html
<link
  rel="preload"
  as="image"
  fetchpriority="high"
  href="https://dims.apnews.com/dims4/default/.../format/webp/quality/90/?url=..."
/>
```

For the video-in-LCP-slot problem: when the top story embeds a Primis video player as the hero, replace it with a static poster image in the initial render. On user click or play-button interaction, swap in the full video player:

```js
document.querySelector(".hero-play-btn").addEventListener("click", () => {
  // load Primis player only on user intent
  const script = document.createElement("script");
  script.src = "https://live.primis.tech/live/liveView.php?...";
  document.body.appendChild(script);
});
```

**Verify**

- Lighthouse → LCP value should decrease. Target: under 2.5 s on desktop.
- DevTools Network → filter by Img → the hero image's request should appear at the earliest start time of any image.
- Lighthouse `lcp-lazy-loaded` audit should pass (no LCP element with `loading="lazy"`).

---

## F4 — Monolithic JS bundle, 78.5% unused at homepage load

**Mechanism**

The entire AP News client-side JavaScript is bundled into one file:

```
https://assets.apnews.com/resource/.../All.min.a44e79227d727121bf50b49c99ddd4cf.gz.js
Resource size: 441.8 KB | Transfer: 109.3 KB (Brotli)
Unused at homepage load: 346.8 KB (78.5%)
Long-task duration on desktop: 405 ms
```

There is no route-based splitting detectable. The bundle filename contains a content hash (`a44e79227...`) so cache-busting is handled correctly, but delivery is unconditional — every page type downloads the same file. Homepage visitors receive and execute code for article rendering, gallery sliders, quiz logic, and search — none of which is present on the homepage route.

**Reproduce**

1. Lighthouse → "Reduce unused JavaScript" audit → `All.min.gz.js` listed with 346.8 KB estimated savings.
2. Lighthouse → "Script treemap" panel (the interactive treemap in the full Lighthouse report HTML) → the entire bundle is one node; no route subtrees visible.
3. DevTools Coverage tab → record a full page load → export CSV → `All.min.gz.js` row shows ~78% red.

**Fix**

This is a **build pipeline change**, not a template patch. The scope depends on your bundler.

**If using Webpack:**

Replace the single entry point with route-specific entry points, and use dynamic `import()` for code that only some routes need:

```js
// webpack.config.js — before (single entry)
entry: { main: './src/index.js' }

// after (per-route entries)
entry: {
  home:    './src/entries/home.js',
  article: './src/entries/article.js',
  gallery: './src/entries/gallery.js',
  search:  './src/entries/search.js',
}
```

The server then includes only the bundle matching the current route:

```html
<!-- homepage template -->
<script src="/js/home.[hash].js" defer></script>

<!-- article template -->
<script src="/js/article.[hash].js" defer></script>
```

Shared vendor code (React, a date library, etc.) goes in a separate chunk loaded on all routes:

```js
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendors: { test: /node_modules/, name: 'vendors', chunks: 'all' }
    }
  }
}
```

**If using a different bundler** (Rollup, esbuild, Vite): the pattern is the same — multiple entry points, shared chunk extraction, server-side template includes the correct entry per route.

**This is a structural change.** Coordinate with your team lead before starting. The primary risk is that server-side route detection and bundle-include logic needs to be consistent — a mismatch (article bundle on homepage, or vice versa) causes a silent regression where route-specific features stop working. Write integration tests per route before starting.

**Target:** each route's own bundle should be under 80 KB resource size. Vendor chunk can be shared and cached separately.

**Verify**

- Lighthouse Coverage audit: homepage route `home.[hash].js` should show unused bytes well below 30%.
- Lighthouse "Reduce unused JavaScript" audit: estimated savings should drop significantly.
- TBT should decrease (less script evaluation time per route). Target: under 300 ms on desktop.
- Run the full site in a staging environment and confirm all interactive features on each route still work after the split.

---

## F5 — Third-party scripts consume 68% of page weight and 9,900 ms of main thread

**Mechanism**

40 distinct third-party origins load on the homepage. All execute upfront, during the critical loading window. Key offenders by main-thread time (desktop Lighthouse `third-parties-insight`):

| Vendor                        | Main thread (ms) | Transfer (KB) | Loading mode       |
| ----------------------------- | ---------------- | ------------- | ------------------ |
| Wunderkind (behavioral popup) | 3,181            | 242.5         | Synchronous, early |
| primis.tech HLS video player  | 2,181            | 3,202.6       | Synchronous, early |
| confiant-integrations.net     | 561              | 217.3         | Synchronous        |
| Quantcast                     | 557              | 259           | Synchronous        |
| Google Tag Manager            | 475              | 334           | Synchronous        |
| Nativo                        | 416              | 989.6         | Synchronous        |

Total identified third-party main-thread time: **~9,900 ms** (desktop). Total third-party transfer: **9,723 KB** (68% of total page transfer).

`html-load.cc` is an unrecognized domain loading 141 KB and executing 110 ms. It does not match any known ad-tech, analytics, or publisher-services vendor. This should be audited immediately — see note below.

**Reproduce**

Lighthouse desktop → "Third-party usage" audit (`third-parties-insight`). Rows are sorted by main-thread blocking time. The full list with transfer sizes is in the audit's items table.

DevTools Network tab → filter by domain: enter each vendor domain to isolate its requests and sizes.

> **Security note:** `html-load.cc` transferring 141 KB from an unrecognized domain is a potential supply-chain injection risk. Before treating it as a performance problem, confirm with whoever manages the GTM container or ad-tech stack whether this domain was intentionally added. If no one can account for it, remove it and monitor for regressions.

**Fix**

There is no single code change here — this requires decisions about which vendors are necessary and what their loading conditions should be. Below are the actionable patterns per vendor type:

**Wunderkind (3,181 ms — highest cost):** This is a behavioral email-capture popup. It has zero content value on first load. Load it after the page is interactive:

```js
// Load Wunderkind only after user has been on the page for 3 seconds
// and only if they are not a subscriber (check cookie/session)
window.addEventListener("load", () => {
  setTimeout(() => {
    if (!userIsSubscriber()) {
      const s = document.createElement("script");
      s.src = "https://tag.bounceexchange.com/...";
      document.body.appendChild(s);
    }
  }, 3000);
});
```

**primis.tech (2,181 ms, 3,202 KB):** The HLS video player library (`hls.min.js`, 544 KB, 91% unused) downloads even when no video ad is in the viewport. Use a facade — show a placeholder, load the real player on IntersectionObserver trigger:

```js
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        // Replace placeholder with actual Primis embed
        loadPrimisPlayer(entry.target);
        observer.unobserve(entry.target);
      }
    });
  },
  { rootMargin: "200px" },
);

document
  .querySelectorAll(".primis-placeholder")
  .forEach((el) => observer.observe(el));
```

**Google Tag Manager:** Move all non-critical tag firing rules to fire on `DOMContentLoaded` or `window.load` instead of immediately. This is configured in the GTM interface, not in code.

**Verify**

- Lighthouse `third-parties-insight`: main-thread time for deferred vendors should drop to near-zero or zero (deferred scripts do not appear in the Lighthouse main-thread snapshot since Lighthouse captures the initial load window).
- TBT on desktop: target under 600 ms after deferring Wunderkind and Primis (those two alone account for ~5,362 ms of the reported 9,900 ms).
- Confirm deferred vendors still fire correctly by manually triggering the interactions that activate them (scroll to Primis slot, wait for Wunderkind timeout).

---

## F6 — Images oversized, no AVIF, quality setting too high

**Mechanism**

AP News uses the DIMS CDN for all editorial images, which supports format, resize, crop, and quality transformations via URL parameters. Despite this capability, three problems appear in the audit:

1. **Oversized delivery:** Lighthouse `image-delivery-insight` flags 12 images with a combined 1,763 KB of potential savings. Images are served at pixel dimensions larger than their CSS display size (e.g., a 1,440×960px image for a slot that renders at ~600px).
2. **No AVIF:** All 30 DIMS images use WebP (`format/webp` in the URL). AVIF offers 20–50% better compression than WebP at equivalent visual quality and is supported by all major browsers as of 2024. No AVIF usage is present anywhere on the homepage.
3. **Quality 90 on thumbnails:** DIMS URLs use `quality/90`. For news thumbnails viewed at small display sizes, quality 75–80 is visually indistinguishable to readers and reduces file size by 15–30%.

A representative DIMS URL from the audit:

```
https://dims.apnews.com/dims4/default/[hash]/2147483647/strip/true/crop/3000x2000+0+0/resize/1440x960/quality/90/format/webp/https://storage.googleapis.com/afs-prod/media/...
```

**Reproduce**

Lighthouse → "Properly size images" audit (`uses-responsive-images`): lists each oversized image with its display size vs. served size, and estimated savings.

DevTools Network → filter by image → click any DIMS request → inspect the URL parameters: `resize/`, `quality/`, `format/`.

**Fix**

All changes are to DIMS URL parameters generated server-side. No JS or CSS changes needed.

**Resize to actual display size:**

The server-side template that generates each `<img>` should pass the rendered display width (from the CSS grid/layout) to the DIMS `resize` parameter. For a 3-column desktop grid where the card renders at 480px wide, the image should be `resize/480x320` (at 1× density) or `resize/960x640` for HiDPI. Use `srcset` to serve different sizes per device pixel ratio:

```html
<img
  src="https://dims.apnews.com/dims4/default/[hash]/strip/true/crop/3000x2000/resize/480x320/quality/80/format/avif/https://storage...."
  srcset="
    https://dims.apnews.com/dims4/default/[hash]/strip/true/crop/3000x2000/resize/480x320/quality/80/format/avif/https://storage.... 480w,
    https://dims.apnews.com/dims4/default/[hash]/strip/true/crop/3000x2000/resize/960x640/quality/80/format/avif/https://storage.... 960w
  "
  sizes="(max-width: 600px) 480px, 960px"
  alt="[story description]"
/>
```

**Switch to AVIF with WebP fallback:**

Change `format/webp` to `format/avif` in the primary URL. Add a `format/webp` srcset as a fallback for older browsers:

```html
<picture>
  <source
    type="image/avif"
    srcset="https://dims.apnews.com/.../format/avif/... 1x"
  />
  <img
    src="https://dims.apnews.com/.../format/webp/..."
    alt="[story description]"
  />
</picture>
```

Verify DIMS supports `format/avif` — given it already handles `format/webp` on-the-fly, AVIF support is standard on modern DIMS/Thumbor instances. Test by requesting a DIMS URL with `format/avif` manually and inspecting the `Content-Type` response header.

**Reduce quality to 80:**

Change `quality/90` → `quality/80` on all non-hero images. Keep `quality/90` on the LCP hero image only.

**Verify**

- Lighthouse `uses-responsive-images`: all flagged images should be resolved (displayed size within 10% of served size).
- Lighthouse `modern-image-formats` (`uses-optimized-images`): AVIF images will not be flagged.
- Estimate total image transfer reduction: the audit flags 1,763 KB of savings from sizing alone. AVIF conversion adds an additional 20–40% reduction on those same images.

---

## F7 — 8 AP custom fonts load with no `font-display` strategy (980 ms FOIT)

**Mechanism**

Lighthouse `font-display-insight` flags 8 AP-branded font files loaded from `assets.apnews.com`. None have a `font-display` descriptor. The worst offender:

```
APW05-Condensed.woff2: 980 ms of invisible text (FOIT — Flash of Invisible Text)
```

Without `font-display`, browsers default to `font-display: auto` which typically implements FOIT — text remains invisible until the custom font is downloaded and applied. This means headlines and body text are invisible for up to 1 second after the page visually renders, then appear in a sudden swap that users see as a jump.

**Reproduce**

Lighthouse → "Font display" audit (`font-display`). Lists all flagged font URLs and their estimated invisible text durations.

In the DevTools Network tab, filter by Font type and sort by size — the AP font files appear there. Their load start time relative to FCP shows the FOIT window.

**Fix**

In the `@font-face` declarations (which are in `All.min.gz.css`), add `font-display: swap` (or `optional` for non-critical fonts):

```css
@font-face {
  font-family: "APW05-Condensed";
  src: url("https://assets.apnews.com/.../APW05-Condensed.woff2")
    format("woff2");
  font-display: swap; /* add this */
}
```

`swap` renders text in a system fallback font immediately, then swaps in the custom font when it loads. This eliminates invisible text at the cost of a font swap. To minimize the swap jump, define a closely-matched fallback stack:

```css
body {
  font-family: "APW05-Condensed", "Arial Narrow", Arial, sans-serif;
}
```

Use the [Font Style Matcher](https://meowni.ca/font-style-matcher/) tool or the `@font-face size-adjust` descriptor to tune the fallback to match the custom font's metrics. This reduces layout shift from the font swap.

For fonts that are purely decorative or not used above the fold, use `font-display: optional` — the browser will use the fallback if the custom font is not available in the first 100 ms, and will not swap later:

```css
@font-face {
  font-family: "APW05-Decorative";
  src: url("...") format("woff2");
  font-display: optional;
}
```

**Verify**

- Lighthouse `font-display` audit: all 8 font entries should pass.
- DevTools Performance → record load → in the Main thread flame chart, look for "Recalculate Style" events after font loads (the swap trigger) — confirm they are present but small.
- CLS score: the font swap may cause a minor CLS event. If CLS increases after adding `swap`, tune the fallback font metrics using `size-adjust` or switch to `optional`.

---

## F8 — Forced synchronous layout from multiple scripts (7,871 ms Style & Layout cost)

**Mechanism**

The Lighthouse `forced-reflow-insight` audit is flagged. Multiple scripts read layout geometry properties (e.g., `offsetWidth`, `getBoundingClientRect`) immediately after writing to the DOM, forcing the browser to synchronously recalculate layout mid-task. This pattern — write DOM → read geometry → write DOM → repeat — is called layout thrashing and is the primary cause of the 7,871 ms Style & Layout cost on desktop.

Top offenders from the audit:

| Script                                            | Forced reflow time             |
| ------------------------------------------------- | ------------------------------ |
| `cdn.jwplayer.com/libraries/8z1I5w4s.js`          | 517.7 ms                       |
| `assets.apnews.com/.../All.min.gz.js` (AP bundle) | 143.9 ms + 123.3 ms = 267.2 ms |
| `securepubads.g.doubleclick.net/pubads_impl.js`   | 152.2 ms                       |
| `apcdp.apnews.com/plugin/library/...`             | 153.1 ms + 89.5 ms = 242.6 ms  |
| `imasdk.googleapis.com/js/sdkloader/ima3.js`      | 143.0 ms + 100.5 ms = 243.5 ms |
| `cdn.cookielaw.org/.../otBannerSdk.js`            | 142.7 ms + 82.5 ms = 225.2 ms  |

**Reproduce**

Lighthouse → "Avoid large layout shifts" (performance) and the `forced-reflow-insight` audit. The audit lists each script with its reflow time.

DevTools Performance tab → record load → in the flame chart, look for yellow "Layout" bars that begin inside a script's call stack (a layout bar inside a JS task, not between tasks, indicates a forced synchronous layout). The "Layout" bars for JW Player will be visually prominent.

**Fix**

**For the AP News first-party bundle (267 ms):** Find the layout-thrashing pattern in the source (if source maps are available in a dev build). The fix is to batch DOM reads before DOM writes using `requestAnimationFrame` or by reading all geometry values first and writing after:

```js
// Bad — read → write → read → write (forces layout twice)
const h1 = el1.offsetHeight; // read → layout forced
el1.style.height = h1 + "px"; // write
const h2 = el2.offsetHeight; // read → layout forced again
el2.style.height = h2 + "px"; // write

// Good — read all, write all (layout forced once)
const h1 = el1.offsetHeight; // read
const h2 = el2.offsetHeight; // read (no layout forced — no write between reads)
el1.style.height = h1 + "px"; // write
el2.style.height = h2 + "px"; // write
```

**For JW Player (517 ms):** This is a third-party library. You cannot change its source. Mitigations:

- Load JW Player lazily (only when a video is in or near the viewport, via IntersectionObserver). The forced reflow only costs time when the library runs — delaying its execution to after page load removes it from the critical window.
- Upgrade to the latest JW Player version; layout-thrashing bugs are commonly fixed in newer releases.

**For Google DoubleClick, IMA SDK, OneTrust:** Same principle as JW Player — lazy loading or deferral prevents their reflows from impacting the page load window.

**Verify**

- DevTools Performance trace: after the fix, the Style & Layout category time should decrease.
- Lighthouse `forced-reflow-insight`: the AP News bundle should no longer appear (or appear with a much lower time) after fixing the first-party thrashing.
- Third-party reflows will persist until those vendors are deferred (F5 fix).

---

## F9 — Warm-load caching is nearly ineffective (1.08% transfer reduction)

**Mechanism**

Cold load: 9,548,467 bytes transferred (271 requests).
Warm load (same session, soft refresh): 9,445,245 bytes transferred (242 requests).
Reduction: **1.08%** — essentially no benefit from having visited once.

Only 3 requests resolved from cache on the warm run. The high-byte resources (JS, CSS, images, third-party scripts) are re-fetched on every load. This suggests:

- The main HTML document likely has a short `max-age` (expected for live news), but so do many static assets.
- `immutable` cache directives may not be set on hashed static assets, causing browsers to issue revalidation requests even for files whose names encode their content.

**Reproduce**

Lighthouse network-requests audit: run cold, record transfer total. Clear nothing. Run again. Compare total bytes. The `lighthouse-network-cold.json` and `lighthouse-network-warm.json` files in this repo are the baseline runs.

DevTools Network tab → reload a page → reload again without clearing cache → compare the "Size" column: entries showing "(from cache)" or "(from memory cache)" vs. re-downloaded entries.

**Fix**

Assets with content-hash filenames (AP News already uses these: `All.min.[hash].gz.js`) should be served with:

```
Cache-Control: public, max-age=31536000, immutable
```

The `immutable` directive tells the browser this URL will never change (because the hash changes when the content changes), so it must not revalidate even at the `max-age` boundary. Without `immutable`, browsers issue a conditional GET (`If-None-Match`) on every visit.

This change is made in the CDN or origin server configuration for `assets.apnews.com`:

**Apache (`.htaccess`):**

```
<FilesMatch "\.[0-9a-f]{8,}\.(js|css)$">
  Header set Cache-Control "public, max-age=31536000, immutable"
</FilesMatch>
```

**Nginx:**

```nginx
location ~* \.[0-9a-f]{8,}\.(js|css)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}
```

**CloudFront / CDN:** Set the origin response header policy to attach `Cache-Control: public, max-age=31536000, immutable` on any response where the URL path contains a content hash.

The main HTML document should retain a short `max-age` (2–5 minutes) or `no-cache` — this is correct behavior for live news and should not be changed.

**Verify**

- DevTools Network tab → "Size" column → hashed JS and CSS files should show "(from disk cache)" on the second visit with no revalidation request.
- Warm-load transfer size should drop significantly — JS (109 KB) and CSS (107 KB) at minimum should be served from cache.
- Lighthouse `uses-long-cache-ttl` audit: hashed static files should no longer appear in the flagged list.

---

## F10 — Accessibility: interactive controls missing accessible names, images without alt

**Mechanism**

Lighthouse Accessibility audit score: 76 (mobile), 79 (desktop). The audit flags:

- **Buttons and links with no accessible name:** Icon-only buttons (e.g., share, bookmark, close buttons) render as `<button>` or `<a>` elements with only an SVG or icon font character inside. Screen readers announce them as "button" with no name — the user cannot determine the button's function.
- **Images without alt attributes:** Several `<img>` elements in the homepage HTML have no `alt` attribute. Screen readers read the filename as a substitute, which is meaningless to listeners.
- **Heading order issues:** Heading levels skip (e.g., `<h1>` followed by `<h3>`, with no `<h2>`), which disrupts screen reader navigation.

**Reproduce**

1. Lighthouse → Accessibility audit → expand each flagged audit item (e.g., `button-name`, `image-alt`, `heading-order`) → each item lists the affected DOM element with a CSS selector and the rule violated.
2. DevTools → Elements panel → find the listed elements directly.
3. Run `axe` browser extension on the homepage for a more detailed interactive report.

**Fix — buttons/links without accessible names:**

```html
<!-- Before: icon-only, no name -->
<button class="share-btn">
  <svg>...</svg>
</button>

<!-- After: aria-label provides the accessible name -->
<button class="share-btn" aria-label="Share this story">
  <svg aria-hidden="true">...</svg>
</button>
```

If the icon is an `<img>`:

```html
<img src="icon-share.svg" alt="Share" />
```

For icon font characters (e.g., using `:before` pseudo-element):

```html
<button class="share-btn" aria-label="Share this story">
  <span class="icon icon-share" aria-hidden="true"></span>
</button>
```

**Fix — images without alt:**

Every `<img>` must have an `alt` attribute. For meaningful images (news photographs):

```html
<img
  src="photo.jpg"
  alt="President Biden signs the infrastructure bill at the White House"
/>
```

For decorative images that convey no information:

```html
<img src="decorative-divider.svg" alt="" />
```

Empty `alt=""` is correct and intentional for decorative images — it tells screen readers to skip the element.

**Fix — heading order:**

Open the page in DevTools → Console → run:

```js
[...document.querySelectorAll("h1,h2,h3,h4,h5,h6")]
  .map((h) => `${h.tagName}: ${h.textContent.trim().substring(0, 50)}`)
  .forEach((s) => console.log(s));
```

This prints the full heading hierarchy. Fix skipped levels by either: changing the heading level in the template, or using `role="heading" aria-level="2"` if the visual style must be preserved independently of the semantic level.

**Verify**

- Lighthouse Accessibility score should increase from 76/79 toward 90+.
- `button-name` audit: 0 violations.
- `image-alt` audit: 0 violations.
- `heading-order` audit: 0 violations.
- Run the page through [WAVE](https://wave.webaim.org/) or axe DevTools extension for a secondary confirmation.

---

## F11 — Minor CLS from late-loading ad and header widget injection

**Mechanism**

Lighthouse reports 2 layout shifts (desktop, CLS: 0.064):

1. `body > div.Page-content` — score 0.064. The entire page content wrapper shifts after initial render. Most likely cause: the video ad slot at the top of the homepage inserts a `<div>` that pushes existing content down after the initial render.
2. `div.Page-header-right` — score 0.00016. Minor shift in the header, likely caused by Zephr (paywall tool) or OneTrust injecting UI after render.

CLS 0.064 is within Google's "needs improvement" band (0.1 is the "poor" threshold). It is not the highest-severity issue but is straightforward to address.

**Reproduce**

Lighthouse → `cls-culprits-insight` audit → lists each shift element with its score.

DevTools Performance tab → enable "Layout Shift Regions" in settings → record a load → layout shifts are shown as blue overlays on the viewport at the moment they occur.

**Fix**

For the ad slot at the top of the page: reserve the ad container's dimensions before the ad loads:

```css
.ad-slot-hero {
  min-height: 250px; /* reserve space matching the expected ad size */
  width: 100%;
}
```

When the ad loads, it fills the reserved space rather than pushing content down. If the ad loads at a different size than reserved, there will still be a small shift — use the most common ad size for the slot (IAB standard: 300×250, 728×90, etc.) as the reserved dimension.

For the header injection: wrap the Zephr/OneTrust header widget in a container with fixed dimensions so that when the widget appears it occupies pre-reserved space.

**Verify**

- Lighthouse CLS value should drop to under 0.01.
- DevTools Performance → Layout Shift Regions: no visible blue overlays after the ad slot is pre-sized.

---

## Domains Considered and Found Clean or Not Applicable

**Service workers / offline support:** AP News is a live news wire. Offline support via service worker cache would deliver stale articles. This is not a meaningful use case for the site's content model. No service worker is present; none is recommended. Not applicable.

**Non-composited animations:** Lighthouse `non-composited-animations` audit returned no flagged items on the desktop run. No CSS or Web Animations API animations were observed running on non-compositor layers at audit time. Clean — no action needed. Note: this audit is load-time only; if animations play after user interaction, test those separately in DevTools.

**HTTP/2 server push / 103 Early Hints:** The site runs on HTTP/2 (confirmed via Network tab protocol column). No `Link: rel=preload` response headers were observed that would trigger H2 push, and no 103 Early Hints responses were detected. This is not a finding — H2 push is largely deprecated in favor of `<link rel="preload">` in HTML, and Early Hints requires CDN and browser support that is still being rolled out. No action needed.

**Web app manifest / PWA installability:** AP News does not appear to be targeting PWA installation. No `manifest.json` reference was observed in the HTML. For a news site, this is a reasonable product decision — most news readers do not install news apps via the browser. Not applicable.

**HTTPS and mixed content:** The site fully serves HTTPS. Lighthouse `is-on-https` audit passes. No mixed-content warnings were observed. Clean.

**Duplicate JavaScript modules:** Lighthouse `duplicated-javascript` audit estimated only ~2 KB of duplicate module code — negligible. No action needed.

---

_The executive-facing version of this report, with findings translated to business risk and opportunity, is in `executive-report.md`._
