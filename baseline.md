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

## Baseline Summary

- Rendering and interactivity are the dominant user-facing problems (very poor LCP, TTI, TBT, and Speed Index).
- Network payload is large for the homepage, and warm-load byte savings are minimal.
- Compression is generally good for text assets, but absolute byte size and resource volume remain too high.
- First-party JS and CSS are shipped as single monolithic bundles with 78.5% and 87.5% unused bytes respectively at homepage load.
- Third-party scripts account for 68% of transferred bytes and ~9,900 ms of main-thread work; most are loaded upfront with no deferral strategy.
- Images use WebP via a capable CDN but are delivered oversized; AVIF is not used.
