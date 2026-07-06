# Baseline

## Scope

- **Primary page:** https://apnews.com/
- **Measurement date:** 2026-07-06
- **Tool:** Lighthouse (same engine used by PageSpeed Insights)
- **Runs:** mobile default, desktop preset
- **Note:** Both runs reported "page loaded too slowly" warnings, so this should be treated as an initial baseline and re-run later for comparison.

## Core Web Vitals (Primary Page)

### Mobile

- **Largest Contentful Paint (LCP):** 54.3 s
- **Cumulative Layout Shift (CLS):** 0.033
- **Interaction/Interactivity proxy:**
  - **Time to Interactive (TTI):** 56.8 s
  - **Total Blocking Time (TBT):** 18,020 ms
  - **Max Potential FID:** 8,460 ms

### Desktop

- **Largest Contentful Paint (LCP):** 17.2 s
- **Cumulative Layout Shift (CLS):** 0.064
- **Interaction/Interactivity proxy:**
  - **Time to Interactive (TTI):** 21.0 s
  - **Total Blocking Time (TBT):** 4,530 ms
  - **Max Potential FID:** 1,530 ms

## PageSpeed Insights Category Scores (Primary Page)

### Mobile

- **Performance:** 25
- **Accessibility:** 76
- **Best Practices:** 54
- **SEO:** 85

### Desktop

- **Performance:** 25
- **Accessibility:** 79
- **Best Practices:** 54
- **SEO:** 85

## Additional Performance Baseline Metrics

### Mobile

- **First Contentful Paint (FCP):** 9.1 s
- **Speed Index:** 22.3 s
- **Server Response Time (root document):** 40 ms

### Desktop

- **First Contentful Paint (FCP):** 3.4 s
- **Speed Index:** 15.7 s
- **Server Response Time (root document):** 40 ms

## Baseline Evidence of Likely Bottlenecks

From Lighthouse opportunities/diagnostics on the primary page:

- **Unused JavaScript:**
  - Mobile potential savings: ~11,660 ms and ~2,480,864 bytes
  - Desktop potential savings: ~2,060 ms and ~3,397,272 bytes
- **Unused CSS:**
  - Mobile potential savings: ~650 ms and ~138,025 bytes
  - Desktop potential savings: ~50 ms and ~136,041 bytes
- **Main-thread work breakdown:** flagged as poor on both mobile and desktop

These bottlenecks align with the very poor LCP, TTI, and TBT values above.

## Networking Baseline (Primary Page)

Network capture source: Lighthouse `network-requests` audit on homepage, with a cold run and a warm run using the same browser profile to approximate soft refresh caching behavior.

### Request volume

- **Cold load requests:** 271
- **Warm run requests:** 242

### Transfer and total resource size

- **Cold load transfer size:** 9,548,467 bytes (about 9.55 MB)
- **Cold load total resource size:** 25,960,060 bytes (about 25.96 MB)
- **Compression reduction (cold):** 63.22%

### Soft refresh and caching effect

- **Warm run transfer size:** 9,445,245 bytes (about 9.45 MB)
- **Transfer reduction vs cold:** 1.08%
- **Cached requests observed (warm):** 3

### JS/CSS vs images

- **JS/CSS**
  - requests: 98
  - transfer: 4,571,886 bytes (47.88% of total transfer)
  - total resource size: 16,684,040 bytes
  - estimated compression reduction: 72.60%

- **Images**
  - requests: 67
  - transfer: 3,455,505 bytes (36.19% of total transfer)
  - total resource size: 3,440,302 bytes
  - estimated compression reduction: -0.44% (effectively no additional content-encoding compression)

### Networking interpretation notes

- Text assets (JS/CSS) appear strongly compressed overall, but remain large in absolute bytes.
- Images account for a large portion of transfer and seem to rely mostly on native image format compression rather than additional content-encoding.
- The warm run shows very limited cache benefit, suggesting constrained cache reuse for this page experience.
