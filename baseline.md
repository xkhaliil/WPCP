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

## Baseline Summary

- Rendering and interactivity are the dominant user-facing problems (very poor LCP, TTI, TBT, and Speed Index).
- Network payload is large for the homepage, and warm-load byte savings are minimal.
- Compression is generally good for text assets, but absolute byte size and resource volume remain too high.
