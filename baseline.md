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
