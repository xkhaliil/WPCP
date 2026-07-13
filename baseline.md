# Baseline

## Scope and Method

- Primary page: https://apnews.com/
- Measurement date: 2026-07-06
- Tools:
  - Lighthouse (mobile default + desktop preset) for performance baseline
  - Mobile runs used Lighthouse throttled mobile emulation (CPU/network throttling as applied by Lighthouse)
  - Lighthouse network-requests audits for cold vs warm networking baseline
- Note: Lighthouse runs reported slow-load warnings, so this is a baseline snapshot and should be re-run later for trend comparison.

## Core Web Vitals and Related UX Metrics (Primary Page)

### Mobile (throttled Lighthouse emulation)

- LCP: 54.3 s
- CLS: 0.033
- TTI: 56.8 s
- TBT: 18,020 ms
- Max Potential FID: 8,460 ms
- FCP: 9.1 s
- Speed Index: 22.3 s
- Server response time (root document): 40 ms

### Desktop

- LCP: 17.2 s
- CLS: 0.064
- TTI: 21.0 s
- TBT: 4,530 ms
- Max Potential FID: 1,530 ms
- FCP: 3.4 s
- Speed Index: 15.7 s
- Server response time (root document): 40 ms

## PageSpeed Category Scores (Primary Page)

### Mobile (throttled Lighthouse emulation)

- Performance: 25
- Accessibility: 76
- Best Practices: 54
- SEO: 85

### Desktop

- Performance: 25
- Accessibility: 79
- Best Practices: 54
- SEO: 85

## Networking Baseline (Primary Page)

Network source: Lighthouse network-requests audit on homepage, with cold run and warm run in the same browser profile to approximate soft refresh behavior.

### Request Count

- Cold load requests: 271
- Warm run requests: 242

### Transfer, Resource Size, and Compression

- Cold transfer size: 9,548,467 bytes (about 9.55 MB)
- Cold total resource size: 25,960,060 bytes (about 25.96 MB)
- Compression reduction (cold): 63.22%

### Soft Refresh and Caching

- Warm transfer size: 9,445,245 bytes (about 9.45 MB)
- Transfer reduction vs cold: 1.08%
- Cached requests observed (warm): 3

### JS/CSS vs Images (Cold Run)

- JS/CSS:
  - requests: 98
  - transfer: 4,571,886 bytes (47.88% of total transfer)
  - resource size: 16,684,040 bytes
  - estimated compression reduction: 72.60%

- Images:
  - requests: 67
  - transfer: 3,455,505 bytes (36.19% of total transfer)
  - resource size: 3,440,302 bytes
  - estimated compression reduction: -0.44% (effectively no extra content-encoding compression)

## Likely Bottleneck Signals Observed

- Unused JavaScript opportunity:
  - mobile potential savings: about 11,660 ms and 2,480,864 bytes
  - desktop potential savings: about 2,060 ms and 3,397,272 bytes
- Unused CSS opportunity:
  - mobile potential savings: about 650 ms and 138,025 bytes
  - desktop potential savings: about 50 ms and 136,041 bytes
- Main-thread work breakdown: flagged poor on mobile and desktop

## Bundle Analysis (Primary Page)

Source: Lighthouse script-treemap-data, network-requests, unused-javascript, unused-css-rules, render-blocking-insight, bootup-time, third-parties-insight, and image-delivery-insight audits from the desktop and mobile runs.

### JavaScript

**Bundle structure:**

- AP News ships one monolithic first-party bundle: `All.min.[hash].gz.js` (441.8 KB resource size, 109.3 KB encoded transfer). No route splitting or component splitting is detectable.
- A small web components polyfill loader (`webcomponents-loader.[hash].gz.js`) is loaded as a separate file.
- Bundle filenames include content hashes, so cache-busting is handled correctly.
- All third-party scripts are loaded separately and not bundled with the first-party code.

**Is this the right decision?**
No. A single all-pages bundle on the homepage delivers code for article pages, galleries, quizzes, and other templates that are not needed on this route. Route splitting would allow loading only the code required for the homepage.

**Unused JavaScript:**

- First-party main bundle: 78.5% unused bytes at homepage load (346.8 KB of 441.8 KB unused)
- Desktop: Lighthouse estimates ~3,318 KB total savings across all scripts if unused JS is removed
- Mobile: Lighthouse estimates ~2,423 KB total savings
- Largest third-party unused JS contributors (desktop): primis.tech video player (hls.min.js: 544 KB resource, 91% unused; prebidVid.min.js: 676 KB resource, 65% unused), s.ntv.io/serve/load.js (930 KB resource, 78% unused), Wunderkind smart-tag (78% unused), Google reCAPTCHA (67% unused)

**Bundle quality:**

- Legacy JavaScript detected (est. 35 KB savings): CoreJS polyfills are being shipped for browser features that modern targets already support. Affected scripts include s.ntv.io/serve/load.js and cdn.jwplayer.com/libraries.
- Duplicated JavaScript: negligible (est. 2 KB savings)
- JS libraries detected: jQuery 3.7.1 (loaded by Wunderkind/BounceExchange, not directly by AP News), core-js polyfills (multiple versions: 3.36.1, 3.36.1, 3.36.1, 3.26.1, 2.6.5 — multiple polyfill versions shipped simultaneously)

**Source maps:**

- Third-party vendors (Viafoura, OneSignal, Sailthru/Sail-Horizon) expose publicly accessible source maps via `sourceMappingURL` headers. Source maps for third-party libraries expose their internal source tree to any visitor.
- AP News's own first-party bundle (`All.min.gz.js`) does not expose a public source map. This is appropriate for production — private source maps should be used for internal debugging only.

**Render-blocking JS:**

- `apcdp.apnews.com/script.js` (AP News CDP/analytics): 141.9 KB, blocks for 1,263 ms
- `experiments.parsely.com/vip-experiments.js`: 9.1 KB, blocks for 339 ms
- `cdn.cookielaw.org/…/OtAutoBlock.js` (OneTrust): 3.3 KB, blocks for 258 ms
- `live.primis.tech/live/liveView.php` (video ad loader): 22 KB, blocks for 436 ms
- Total JS render-blocking penalty: ~2,296 ms (desktop)

### CSS

**Bundle structure:**

- AP News ships one monolithic stylesheet: `All.min.[hash].gz.css` (107 KB transfer). No route splitting or critical-CSS extraction is detectable.
- Third-party CSS loaded separately: Viafoura community CSS (13.1 KB), OneTrust banner inline CSS (22 KB), JW Player inline CSS (12.4 KB), Primis video CSS, Riverdrop partner CSS.
- The main stylesheet is loaded synchronously in `<head>` without `media` queries or async loading, making it render-blocking.

**Is this the right decision?**
No. A single stylesheet for all routes is loaded even though the vast majority of its rules are not applicable on the homepage. Extracting critical above-the-fold CSS and deferring the rest would eliminate the render-blocking penalty and reduce parse work.

**Unused CSS:**

- Main bundle: 87.5% unused (93.6 KB wasted of 107 KB total)
- OneTrust inline CSS: 85.9% unused (18.9 KB of 22 KB wasted)
- Viafoura CSS: 97.8% unused (12.8 KB of 13.1 KB wasted)
- JW Player inline CSS: 86.2% unused (10.7 KB of 12.4 KB wasted)
- Desktop estimated CSS savings: 133 KB
- The main CSS is also the single most expensive render-blocker on the page: 1,882 ms blocking penalty (desktop)

### Images

**Formats and sizing:**

- AP News uses the DIMS image CDN (`dims.apnews.com`) for all editorial images, supporting on-the-fly crop, resize, and format transformation via URL parameters.
- 35 DIMS images observed on desktop run: 30 using WebP (`format/webp` in URL), 5 without an explicit format parameter.
- No AVIF usage detected anywhere on the homepage.
- Non-DIMS images: 22 JPEG, 10 SVG, 9 GIF, 8 PNG (largely third-party UI assets, tracking pixels, and ad creatives).

**Is this the right decision?**
Partially. Using WebP via DIMS for editorial images is good practice. However, AVIF offers 20–50% better compression than WebP at equivalent quality and is now supported by all major browsers. Not using AVIF is a missed optimization. Additionally, despite having a capable image CDN, images are still delivered at sizes larger than their display context (see below).

**Is the full-resolution file exposed?**
The DIMS URLs contain a `?url=` query parameter pointing to the original source asset in AP's media storage. While the DIMS service applies transformations, the encoded origin URL in query parameters may expose the location of the full-resolution asset to any user who inspects the request URL.

**Image delivery issues:**

- Lighthouse image-delivery-insight flags 12 images for a total of 1,763 KB of potential savings (desktop).
- Images are being served at pixel dimensions larger than their rendered display size (e.g., 1,440×960 images served where the display element is smaller), indicating the DIMS `resize` parameters are not consistently set to match actual CSS display dimensions.
- Quality setting of 90 (`quality/90`) is on the higher end; reducing to 75–80 on news thumbnails is typically imperceptible to readers and would reduce file size further.

### Third-Party Resources

**Inventory (desktop, sorted by main-thread time):**

| Vendor                         | Category                             | Main Thread (ms) | Transfer (KB) | Load Timing      |
| ------------------------------ | ------------------------------------ | ---------------- | ------------- | ---------------- |
| Wunderkind                     | Behavioral marketing / email capture | 3,181            | 242.5         | Upfront, early   |
| primis.tech                    | Video ad platform                    | 2,181            | 3,202.6       | Upfront, early   |
| confiant-integrations.net      | Ad quality / security                | 561              | 217.3         | Upfront          |
| Quantcast                      | Audience measurement                 | 557              | 259           | Upfront          |
| Google Tag Manager             | Tag management container             | 475              | 334           | Upfront          |
| LongTail Ad Solutions (Nativo) | Native advertising                   | 416              | 989.6         | Upfront          |
| Viafoura                       | Community / comments                 | 313              | 318.6         | Upfront          |
| Optanon / OneTrust             | Cookie consent (GDPR)                | 289              | 371           | Early, blocking  |
| pub.network                    | Header bidding (prebid.js)           | 275              | 441.8         | Upfront          |
| Google / DoubleClick Ads       | Display advertising                  | 249              | 619           | Upfront          |
| Other Google APIs / SDKs       | (Various via GTM)                    | 239              | 791           | Upfront          |
| Kameleoon                      | A/B testing                          | 168              | 37.9          | Blocking (early) |
| Google CDN (JW Player)         | Video player library                 | 140              | 375.6         | Upfront          |
| dianomi                        | Content monetization                 | 125              | 41.8          | Upfront          |
| html-load.cc                   | Unknown (unrecognized domain)        | 110              | 141           | Upfront          |
| Facebook SDK                   | Social login / sharing               | 101              | 83.5          | Upfront          |
| Amazon Ads                     | Advertising                          | 61               | 94.9          | Upfront          |
| permutive.app                  | Audience data platform               | 42               | 144.2         | Upfront          |
| Connatix                       | Video content                        | 28               | 52.2          | Upfront          |
| OneSignal                      | Push notifications                   | 27               | 46.2          | Upfront          |

- Total identified third-party main-thread time: approximately 9,900 ms (desktop)
- Total third-party transfer: 9,723 KB (68% of total 14,280 KB desktop transfer)
- 40 distinct third-party origins observed (desktop)
- OneTrust OtAutoBlock.js and Kameleoon engine.js are render-blocking
- primis.tech delivers 3,202 KB of video ad code on first load, 91% of which (HLS.js) is immediately unused

**Inappropriate or suspicious vendors:**

- `html-load.cc` — unrecognized domain, not a known publishing or ad-tech vendor. Loads 141 KB and executes 110 ms on the main thread. Should be investigated and removed if unauthorized.
- Wunderkind — behavioral popup/email capture tool with the highest main-thread cost (3,181 ms). Non-essential to content consumption; could be deferred until after page becomes interactive.
- primis.tech — video ad platform downloads 3.2 MB upfront even when no video ad is in the viewport. A facade or intersection-observer–based lazy load would eliminate most of this cost.

## Coverage Analysis (Primary Page)

Source: Lighthouse unused-css-rules, unused-javascript, forced-reflow-insight, and render-blocking-insight audits from the desktop run.

### Critical CSS

AP News does **not** extract critical CSS. The single monolithic stylesheet (`All.min.[hash].gz.css`, 107 KB) is loaded synchronously in `<head>` with no `media` attribute, no `rel="preload"` + `onload` trick, and no inline `<style>` tag containing above-the-fold rules. The browser must fully download and parse the entire stylesheet before it can render any visible content. This produces a 1,882 ms render-blocking penalty on desktop.

No critical-CSS extraction tooling (e.g., Critters, Penthouse, or equivalent) appears to be in the build pipeline.

### Unused CSS

| Source                                                           | Transfer (KB) | Wasted (KB) | Wasted % |
| ---------------------------------------------------------------- | ------------- | ----------- | -------- |
| `assets.apnews.com/.../All.min.gz.css` (first-party main bundle) | 107           | 93.6        | 87.5%    |
| OneTrust banner inline CSS (injected by `otBannerSdk.js`)        | 22            | 18.9        | 85.9%    |
| `cdn.viafoura.net/181...css` (community widget)                  | 13.1          | 12.8        | 97.8%    |
| JW Player inline CSS (injected by player scripts)                | 12.4          | 10.7        | 86.2%    |

- Desktop estimated savings from unused CSS: **133 KB**
- The primary cause for all entries is the same: each stylesheet is a single all-pages bundle shipped on a route that only needs a small subset of its rules.

### Unused JavaScript

**First-party:**

- `All.min.[hash].gz.js` (AP News own bundle): 441.8 KB resource size, **346.8 KB (78.5%) unused** at homepage load
- `apcdp.apnews.com/script.js` (AP CDP/analytics): 141.2 KB, ~35.4% unused
- `apcdp.apnews.com/plugin/library/...` and `apcdp.apnews.com/plugin/plugin/...`: combined ~56–65% unused

**Third-party by category:**

| Category                                      | Estimated Wasted (KB, desktop) | Top offenders                                                                                            |
| --------------------------------------------- | ------------------------------ | -------------------------------------------------------------------------------------------------------- |
| Ad / Video scripts                            | ~1,591                         | primis HLS.js (91% unused), primis liveVideo (58%), primis prebid (66%), s.ntv.io (78–82%), pubads (69%) |
| Other 3P (GTM, Viafoura, apcdp, html-load.cc) | ~816                           | html-load.cc (66%), Viafoura (50%), GTM (40%), apcdp (35–72%)                                            |
| Marketing / Analytics                         | ~203                           | BounceExchange smart-tag (75%), OneSignal (69%), Kameleoon (44%)                                         |
| Social                                        | ~54                            | Facebook SDK (69%)                                                                                       |
| reCAPTCHA                                     | ~251                           | Google reCAPTCHA (67%) — loaded but mostly unused on homepage                                            |

- **Desktop total unused JS savings: ~3,318 KB** (Lighthouse estimate)
- **Mobile total unused JS savings: ~2,423 KB** (Lighthouse estimate)
- The dominant cause is upfront loading of ad-stack and video-player scripts that are either not in the current viewport or not needed on the homepage at all.

## Performance Flame Chart Analysis (Primary Page)

Source: Lighthouse long-tasks, mainthread-work-breakdown, and forced-reflow-insight audits from desktop and mobile runs. Note: a full DevTools Performance recording was not captured as a separate artifact; findings below are derived from Lighthouse-reported main-thread data.

### Main-Thread Work Summary

| Category                    | Desktop (ms) | Mobile (ms) |
| --------------------------- | ------------ | ----------- |
| Script Evaluation           | 12,292       | 31,492      |
| Style & Layout              | 7,871        | 4,550       |
| Other                       | 5,325        | 11,023      |
| Script Parse & Compile      | 978          | 3,512       |
| Rendering (paint/composite) | 773          | 2,152       |
| Parse HTML & CSS            | 234          | 1,045       |
| Garbage Collection          | 123          | 518         |
| **Total**                   | **~27,596**  | **~54,292** |

### Long Tasks (Dropped / Skipped Frames)

A "long task" is any main-thread task exceeding 50 ms. Each long task blocks the browser from producing a frame at 60fps (16.7 ms/frame budget), causing dropped or skipped frames for any concurrent scroll or interaction.

**Desktop: 20 long tasks, 5,846 ms total blocked time**

| Script                                                   | Duration (ms) | Start Time (ms into load) |
| -------------------------------------------------------- | ------------- | ------------------------- |
| primis.tech liveVideo.php (video ad)                     | 1,532         | 18,154                    |
| `assets.apnews.com/.../All.min.gz.js` (AP News bundle)   | 405           | 7,712                     |
| Unattributable                                           | 367           | 16,578                    |
| `ssl.p.jwpcdn.com/.../provider.hlsjs.js` (JW Player HLS) | 356           | 20,504                    |
| `cdn.confiant-integrations.net/.../wrap.js` (ad quality) | 302           | 10,779                    |
| `securepubads.g.doubleclick.net/.../pubads_impl.js`      | 287           | 16,196                    |
| `s.ntv.io/serve/load.js` (Nativo)                        | 256           | 7,226                     |
| `s.ntv.io/serve/load.js` (Nativo, second task)           | 230           | 11,224                    |
| `www.googletagmanager.com/gtm.js`                        | 227           | 8,147                     |
| `cdn.viafoura.net/vf-v2.js`                              | 225           | 16,996                    |

**Mobile: 20 long tasks, 19,006 ms total blocked time (~3.25× worse than desktop)**

Top mobile offenders include Wunderkind `cjs_min_...js`, unattributable tasks, s.ntv.io (×2), Google Tag, and AP News own bundle.

### Are Dropped Frames Excessive or Unexpected?

Yes — both in frequency and severity. Key observations:

- **Excessive:** 20 long tasks over a 22+ second load window means the main thread is unavailable to produce frames or respond to input for about 26% of the entire page-load period on desktop, and far more on mobile.
- **Unexpected:** The page's own content (news stories, images) is simple and static. The dropped-frame budget is consumed almost entirely by third-party ad, video, and marketing scripts that are not directly serving content to the reader.
- The single 1,532 ms task from primis.tech alone would drop approximately 91 consecutive frames at 60fps.
- On mobile, Script Evaluation alone (31,492 ms) consumes more main-thread time than the entire desktop trace.

### Forced Synchronous Layout (Style & Layout: 7,871 ms)

The `forced-reflow-insight` audit is flagged. Multiple scripts call DOM geometry APIs (`offsetWidth`, `getBoundingClientRect`, etc.) immediately after a DOM mutation, forcing the browser to synchronously recalculate layout before completing the current task. This is a major contributor to the 7,871 ms Style & Layout cost.

Top forced-reflow offenders (by individual reflow time):

| Script                                                 | Reflow (ms)   |
| ------------------------------------------------------ | ------------- |
| `cdn.jwplayer.com/libraries/8z1I5w4s.js`               | 517.7         |
| `apcdp.apnews.com/plugin/library/...`                  | 153.1         |
| `securepubads.g.doubleclick.net/pubads_impl.js`        | 152.2         |
| `imasdk.googleapis.com/js/sdkloader/ima3.js`           | 143.0 + 100.5 |
| `cdn.cookielaw.org/.../otBannerSdk.js` (OneTrust)      | 142.7 + 82.5  |
| `assets.apnews.com/.../All.min.gz.js` (AP News bundle) | 143.9 + 123.3 |
| `apcdp.apnews.com/plugin/library/...`                  | 89.5          |
| `cdn.viafoura.net/vf-v2.js`                            | 79.5          |
| `ssl.p.jwpcdn.com/.../jwplayer.core.controls.js`       | 77.8          |

The JW Player library alone triggers a single 517 ms forced reflow — a frame-budget violation of ~30 frames. The AP News first-party bundle also triggers two separate forced reflows totaling ~267 ms, indicating layout-thrashing patterns in its own JS.

## Layers and Animations (Primary Page)

Source: Lighthouse non-composited-animations, layout-shifts, cls-culprits-insight, font-display-insight audits, and forced-reflow-insight from the desktop run. Note: a live DevTools Layers panel recording was not captured as a separate artifact; paint layer counts and composition triggers are inferred from Lighthouse signals.

### Do Animations Feel Janky?

Lighthouse's `non-composited-animations` audit found **no flagged animations** in the desktop snapshot — the items array is empty. This means Lighthouse did not observe any CSS or Web Animations API animations that were demoted to the main thread at audit time.

However, this result is load-time only. Given the 5,846 ms of long tasks on desktop and 19,006 ms on mobile during load, **any animation playing during the load phase** (loading spinners, skeleton screens, scroll-triggered fade-ins) would unavoidably drop frames because the main thread is already saturated. The absence of compositing violations does not mean the page animates smoothly — it means no animations were observed running during the crawl window.

First-frame jank is likely: fonts are loading without a `font-display` descriptor, so the browser either renders invisible text (FOIT) or swaps fallback-to-custom-font in the first few seconds. Lighthouse flags **8 AP custom fonts** with no `font-display` strategy, with the worst font (`APW05-Condensed.woff2`) incurring 980 ms of text invisibility. This swapover manifests as a visual flicker/jump that users perceive as first-frame jank.

### Animation Triggers (Layout, Paint, or Composition)

Without a live DevTools recording showing the full animation timeline, the following is inferred from the forced-reflow data:

- **Layout triggers are dominant.** The forced-reflow audit shows dozens of layout recalculations triggered by third-party scripts. Any CSS animation or JS-driven animation that reads layout values inside its animation loop (e.g., reading `scrollTop` and then setting `style.top`) would trigger layout on every frame — the most expensive kind of animation.
- **Paint triggers are likely.** The `paintCompositeRender` category consumes 773 ms on desktop / 2,152 ms on mobile, suggesting non-trivial paint work. Third-party ad iframes frequently trigger paint by updating their content.
- **Composition:** No non-composited animation violations detected, but high rendering cost (2,152 ms on mobile) suggests either many paint layers or large repaint areas from ad iframe refreshes.

### Paint Layers

Exact paint layer count requires a live DevTools Layers recording. From available data:

- Ad iframes (Google DoubleClick, Nativo, primis) each typically create their own stacking context, effectively forcing at least one GPU layer per ad slot. With 10+ ad positions on the AP News homepage, this alone accounts for significant layer count.
- The JW Player video widget creates its own layer for compositing the video surface.
- The OneTrust consent banner and Wunderkind overlay are absolutely positioned and typically create additional layers.
- No evidence of deliberate `will-change` or `transform: translate3d(0,0,0)` layer-promotion hacks was observed in the Lighthouse data. The non-composited-animations audit being clean suggests AP News's own CSS does not force unnecessary GPU layers via these tricks.

### Layout Shifts

Lighthouse detected **2 layout shifts** (desktop, CLS: 0.064):

1. `body > div.Page-content` (score: 0.064) — the entire main content wrapper shifts after initial render. The most likely cause is the late-loading video ad slot in the hero area, which inserts a `<div>` that pushes existing content down before the ad finishes loading.
2. `div.Page-header-right` ("Sign in / Show Search" area, score: 0.00016) — a minor shift in the header, likely caused by Zephr (paywall/subscription tool) or OneTrust injecting UI into the header after initial render.

## Baseline Summary

- Rendering and interactivity are the dominant user-facing problems (very poor LCP, TTI, TBT, and Speed Index).
- Network payload is large for the homepage, and warm-load byte savings are minimal.
- Compression is generally good for text assets, but absolute byte size and resource volume remain too high.
- First-party JS and CSS are shipped as single monolithic bundles with 78.5% and 87.5% unused bytes respectively at homepage load.
- Third-party scripts account for 68% of transferred bytes and ~9,900 ms of main-thread work; most are loaded upfront with no deferral strategy.
- No critical CSS extraction is in place; the entire 107 KB stylesheet blocks first render for ~1,882 ms.
- 20 long tasks totaling 5,846 ms on desktop (19,006 ms mobile) drop frames throughout the load window; the primary cause is third-party ad/video/marketing scripts executing upfront.
- Multiple scripts trigger forced synchronous layout; the JW Player library alone causes a 518 ms forced reflow.
- Images use WebP via a capable CDN but are delivered oversized; AVIF is not used.
- 8 AP custom fonts ship with no `font-display` strategy, causing up to 980 ms of invisible text (FOIT) and visible font swap on load.
