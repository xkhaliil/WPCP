# Findings

This set is cleaned so each finding is independently observable and scoped to one primary problem.

## Prioritization System

I will use a custom **Priority Heat Score (PHS-4)** system for corrective findings.

- **Impact (1-5):** how much the issue hurts users
- **Mobile Severity (1-5):** how much worse it is under throttled mobile conditions
- **Breadth (1-5):** how many metrics or user experiences the issue affects
- **Effort (1-5):** how practical the fix is, where 5 means easiest

**Formula:** `PHS = Impact + Mobile Severity + Breadth + Effort`

The highest possible score is 20. Higher scores mean higher priority.

## Corrective Findings

## 1) Initial page render is significantly delayed

- Type: Corrective
- Ratings: Impact 5, Mobile Severity 4, Breadth 4, Effort 3
- Priority Heat Score: 16/20
- Observable signal: FCP 9.1 s (mobile), 3.4 s (desktop); Speed Index 22.3 s (mobile), 15.7 s (desktop)
- How this affects users:
  Users wait too long before meaningful content appears, which increases bounce risk.
- Metric(s) affected:
  FCP, Speed Index, overall Performance score
- Most likely cause:
  Heavy critical path and blocking resources before first paint.
- Likely solution:
  Reduce render-blocking work, inline minimal critical CSS, and defer non-critical scripts/styles.

## 2) Initial visual load (main content) is significantly delayed

- Type: Corrective
- Ratings: Impact 5, Mobile Severity 5, Breadth 5, Effort 2
- Priority Heat Score: 17/20
- Observable signal: LCP 54.3 s (mobile), 17.2 s (desktop)
- How this affects users:
  The page appears incomplete for too long; users may leave before seeing the main story.
- Metric(s) affected:
  LCP, Speed Index, Performance score
- Most likely cause:
  LCP element not prioritized enough and above-the-fold competition from non-critical assets.
- Likely solution:
  Prioritize LCP candidate (preload/fetch priority), reduce above-the-fold asset contention, and optimize hero media.

## 3) Initial page functionality is significantly delayed

- Type: Corrective
- Ratings: Impact 5, Mobile Severity 5, Breadth 5, Effort 2
- Priority Heat Score: 17/20
- Observable signal: TTI 56.8 s (mobile), 21.0 s (desktop); TBT 18,020 ms (mobile), 4,530 ms (desktop)
- How this affects users:
  Page appears loaded but interaction is delayed or janky (scroll/tap/menu responsiveness).
- Metric(s) affected:
  TBT, TTI, Max Potential FID, Performance score
- Most likely cause:
  Too much JavaScript main-thread work, including third-party execution.
- Likely solution:
  Reduce third-party JS, split bundles, lazy-load non-critical features, and break long tasks.

## 4) Image loading order does not consistently match user need

- Type: Corrective
- Ratings: Impact 4, Mobile Severity 4, Breadth 3, Effort 3
- Priority Heat Score: 14/20
- Observable signal: 67 image requests and 3.46 MB image transfer on cold load; poor LCP/Speed Index together indicate image priority issues
- How this affects users:
  High-value visual content can be delayed while lower-priority imagery also competes for bandwidth.
- Metric(s) affected:
  LCP, Speed Index, image transfer budget
- Most likely cause:
  Weak image prioritization and large media volume loading too early.
- Likely solution:
  Prioritize hero/first-story images, lazy-load below-the-fold images, and enforce responsive image sizing.

## 5) Delayed ad and third-party slots hurt perceived stability

- Type: Corrective
- Ratings: Impact 4, Mobile Severity 4, Breadth 4, Effort 2
- Priority Heat Score: 14/20
- Observable signal: very high third-party script/network activity during early render; low Best Practices score (54)
- How this affects users:
  Layout feels unstable or "still loading" for too long, reducing trust and readability.
- Metric(s) affected:
  LCP, TBT, Speed Index, Best Practices
- Most likely cause:
  Ad/third-party scripts injected too early in the critical rendering path.
- Likely solution:
  Move non-critical ad work later, reserve stable placeholders, and gate third-party script execution.

## 6) Accessibility baseline indicates semantic/control gaps

- Type: Corrective
- Ratings: Impact 3, Mobile Severity 3, Breadth 3, Effort 3
- Priority Heat Score: 12/20
- Observable signal: Accessibility score 76 (mobile), 79 (desktop)
- How this affects users:
  Assistive technology users may struggle with controls, labels, or page structure clarity.
- Metric(s) affected:
  Accessibility score
- Most likely cause:
  Missing accessible names/labels and structural semantics inconsistencies.
- Likely solution:
  Audit interactive controls and headings, ensure accessible names/labels/alt text, and validate with accessibility tooling.

## 7) Homepage payload size is too large

- Type: Corrective (Additional)
- Ratings: Impact 5, Mobile Severity 5, Breadth 5, Effort 2
- Priority Heat Score: 17/20
- Observable signal: 9.55 MB transferred, 25.96 MB total resource size on cold load
- How this affects users:
  Users on mobile networks face slower loads and higher data usage costs.
- Metric(s) affected:
  Transfer size, resource size, FCP/LCP/Speed Index, Performance
- Most likely cause:
  Heavy combined JS/CSS/media footprint on initial navigation.
- Likely solution:
  Set page-weight budgets, remove unused code/media, and stage non-critical downloads after first render.

## 8) Warm-load cache benefit is weak in transferred bytes

- Type: Corrective (Additional)
- Ratings: Impact 4, Mobile Severity 4, Breadth 4, Effort 3
- Priority Heat Score: 15/20
- Observable signal: only 1.08% transfer reduction from cold to warm run (9.55 MB to 9.45 MB)
- How this affects users:
  Repeat visits and soft refreshes remain expensive instead of feeling noticeably faster.
- Metric(s) affected:
  Warm transfer size, cache efficiency, perceived repeat-load speed
- Most likely cause:
  Limited reusable cache footprint for high-byte resources and/or aggressive cache-busting dynamics.
- Likely solution:
  Improve cacheability of stable assets (immutable hashed files, stronger TTL policy) and reduce dynamic high-byte responses.

## Good Findings

## 9) Compression on text assets is strong

- Type: Good
- Observable signal: overall cold compression reduction 63.22%; JS/CSS compression reduction 72.60%
- Why this is good for users:
  Smaller transfer for text resources improves bandwidth efficiency and helps loading speed.
- Metric(s) supported:
  Transfer size efficiency, network performance baseline
- Keep doing:
  Maintain Brotli/gzip delivery and continue compression checks in CI/performance audits.

## 10) Some caching and request reuse behavior is present

- Type: Good
- Observable signal: request count drops from 271 (cold) to 242 (warm); warm run reports cached requests
- Why this is good for users:
  Repeat loads avoid part of the request overhead.
- Metric(s) supported:
  Warm request volume and cache-hit behavior
- Keep doing:
  Keep cache controls for reusable assets and extend to more high-byte resources.

## Mobile-Specific Findings

## 11) Mobile throttling exposes severe first-load delay on the homepage

- Type: Corrective
- Ratings: Impact 5, Mobile Severity 5, Breadth 5, Effort 2
- Priority Heat Score: 17/20
- Observable signal: Mobile LCP 54.3 s, Mobile TTI 56.8 s, Mobile TBT 18,020 ms, Mobile Speed Index 22.3 s
- How this affects users:
  On slower mobile conditions, the homepage feels unusable for a long time and content arrives far too late.
- Metric(s) affected:
  Mobile LCP, TTI, TBT, Speed Index, Performance score
- Most likely cause:
  Heavy render-blocking work and large above-the-fold payload under throttled mobile conditions.
- Likely solution:
  Prioritize the mobile hero content, defer non-critical assets, and reduce JavaScript execution on the first viewport.

## 12) Mobile network payload is too heavy for throttled conditions

- Type: Corrective
- Ratings: Impact 5, Mobile Severity 5, Breadth 5, Effort 2
- Priority Heat Score: 17/20
- Observable signal: Mobile cold transfer 9.55 MB, 271 requests, and only 1.08% warm-load transfer reduction
- How this affects users:
  Mobile users spend too long waiting and consume more data, especially on constrained connections.
- Metric(s) affected:
  Mobile request count, transfer size, resource size, cache efficiency, perceived repeat-load speed
- Most likely cause:
  Too many resources are fetched on the first mobile visit, and the page does not reuse enough bytes on repeat loads.
- Likely solution:
  Reduce mobile request fan-out, trim initial bytes, and improve cache reuse for stable assets.

## Bundle Findings

## 13) First-party JavaScript is a monolithic bundle with 78.5% unused bytes at homepage load

- Type: Corrective (Bundle)
- Ratings: Impact 4, Mobile Severity 4, Breadth 4, Effort 3
- Priority Heat Score: 15/20
- Observable signal: `All.min.[hash].gz.js` is 441.8 KB resource size with 346.8 KB (78.5%) unused at page load; no route splitting or code splitting detected; desktop est. 3,318 KB total unused JS savings; mobile est. 2,423 KB
- How this affects users:
  The browser parses and compiles a large JS bundle upfront that contains code for article pages, galleries, and other templates never used on the homepage, consuming main-thread time before the page is interactive.
- Metric(s) affected:
  TBT, TTI, Max Potential FID, Performance score
- Most likely cause:
  A single monolithic Webpack (or equivalent) bundle is produced for all pages and served unconditionally, with no route-based or component-based code splitting.
- Likely solution:
  Implement route splitting so the homepage only loads homepage-required code. Move article, gallery, quiz, and search-page code into separate chunks loaded on demand.

## 14) Main CSS bundle is monolithic, 87.5% unused, and render-blocking

- Type: Corrective (Bundle)
- Ratings: Impact 4, Mobile Severity 4, Breadth 3, Effort 3
- Priority Heat Score: 14/20
- Observable signal: `All.min.[hash].gz.css` transfers 107 KB with 93.6 KB (87.5%) wasted; it is loaded synchronously in `<head>` and contributes a 1,882 ms render-blocking penalty (desktop); Viafoura CSS 97.8% unused, OneTrust inline CSS 85.9% unused
- How this affects users:
  The browser is blocked from rendering anything visible until a stylesheet mostly full of irrelevant rules has fully downloaded and parsed, directly delaying FCP and LCP.
- Metric(s) affected:
  FCP, LCP, Speed Index, render-blocking budget
- Most likely cause:
  A single catch-all stylesheet is produced for all page templates and served synchronously in every page's `<head>`, with no critical-CSS extraction or route-based CSS splitting.
- Likely solution:
  Extract the critical above-the-fold CSS and inline it in `<head>`; load the remainder asynchronously with `rel="preload"` + `onload` or a media-query trick. Split route-specific styles into separate files loaded only on relevant pages.

## 15) Third-party scripts are loaded upfront with no deferral and dominate main-thread time

- Type: Corrective (Bundle / Third-party)
- Ratings: Impact 5, Mobile Severity 5, Breadth 5, Effort 2
- Priority Heat Score: 17/20
- Observable signal: 40 distinct third-party origins loaded on first visit; Wunderkind 3,181 ms main thread, primis.tech 2,181 ms + 3,202 KB transfer (91% of HLS.js unused), confiant 561 ms, Quantcast 557 ms, GTM 475 ms; total identified third-party main-thread time ≈ 9,900 ms; OneTrust OtAutoBlock.js and Kameleoon engine.js are render-blocking; `html-load.cc` is an unrecognized domain executing 110 ms on main thread
- How this affects users:
  Nearly all third-party advertising, analytics, A/B testing, video, and marketing scripts compete for the main thread from the moment the page starts loading, preventing the browser from processing the actual page content and making the page non-interactive for an extended period.
- Metric(s) affected:
  TBT, TTI, FCP, LCP, Max Potential FID, Best Practices score
- Most likely cause:
  Scripts are injected via synchronous `<script>` tags or Google Tag Manager triggers that fire immediately on page load with no `async`/`defer` discipline and no facade or intersection-observer gating for heavy widgets.
- Likely solution:
  Audit and categorize all third-party scripts by necessity and timing. Defer non-critical scripts (Wunderkind, primis.tech video ads, dianomi, Facebook SDK) until after first user interaction or `requestIdleCallback`. Use facades for the video ad player and comments widget. Remove or investigate `html-load.cc` — it is an unrecognized domain and should not be executing code on the page.

## 16) Images are served oversized for their display context despite using a capable CDN

- Type: Corrective (Bundle / Image Delivery)
- Ratings: Impact 4, Mobile Severity 4, Breadth 3, Effort 3
- Priority Heat Score: 14/20
- Observable signal: Lighthouse image-delivery-insight flags 12 images with 1,763 KB total savings (desktop); DIMS images served at 1,440×960 or 1,440×1,080 when displayed at smaller sizes; quality/90 used throughout; no AVIF format detected on any image
- How this affects users:
  Browsers download significantly more image data than is ever displayed, increasing total transfer size and delaying LCP for the hero image.
- Metric(s) affected:
  LCP, image transfer budget, total page weight
- Most likely cause:
  The DIMS CDN supports parameterized resizing but the resize dimensions in generated URLs are not consistently matched to the actual CSS display dimensions. A single large size is used across multiple breakpoints instead of responsive `srcset` with per-breakpoint dimensions.
- Likely solution:
  Set DIMS resize parameters to match actual rendered dimensions per breakpoint. Use `srcset` and `sizes` attributes on `<img>` elements to let the browser select the correct variant. Switch the preferred format to AVIF (add `format/avif` as the DIMS format parameter with WebP as a fallback) for additional 20–50% size reduction over WebP.

## Priority Order (Corrective)

1. Fix interactivity delays from JavaScript main-thread work, including third-party script deferral (Findings 3 and 15)
2. Fix LCP and initial visual priority (Finding 2)
3. Reduce total payload size and resource competition, including monolithic JS/CSS bundles (Findings 7, 4, 13, and 14)
4. Improve repeat-load cache byte savings (Finding 8)
5. Fix oversized image delivery and adopt AVIF (Finding 16)
6. Stabilize ad/third-party rendering and improve accessibility compliance (Findings 5 and 6)
7. Address mobile-specific first-load delay and payload pressure (Findings 11 and 12)

## Final Score

- Corrective findings scored with PHS-4: 16, 17, 17, 14, 14, 12, 17, 15, 17, 17, 15, 14, 17, 14
- Average corrective score: 15.4/20
- Final score as a percentage: 77/100
