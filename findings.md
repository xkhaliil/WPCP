# Findings

This set is cleaned so each finding is independently observable and scoped to one primary problem.

## Corrective Findings

## 1) Initial page render is significantly delayed

- Type: Corrective
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
- Observable signal: Mobile cold transfer 9.55 MB, 271 requests, and only 1.08% warm-load transfer reduction
- How this affects users:
  Mobile users spend too long waiting and consume more data, especially on constrained connections.
- Metric(s) affected:
  Mobile request count, transfer size, resource size, cache efficiency, perceived repeat-load speed
- Most likely cause:
  Too many resources are fetched on the first mobile visit, and the page does not reuse enough bytes on repeat loads.
- Likely solution:
  Reduce mobile request fan-out, trim initial bytes, and improve cache reuse for stable assets.

## Priority Order (Corrective)

1. Fix interactivity delays from JavaScript main-thread work (Finding 3)
2. Fix LCP and initial visual priority (Finding 2)
3. Reduce total payload size and resource competition (Findings 7 and 4)
4. Improve repeat-load cache byte savings (Finding 8)
5. Stabilize ad/third-party rendering and improve accessibility compliance (Findings 5 and 6)
6. Address mobile-specific first-load delay and payload pressure (Findings 11 and 12)
