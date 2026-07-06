# Findings

## 1) Extremely slow Largest Contentful Paint on the homepage

- **How this affects users:**
  The main content appears very late, so users see a blank or incomplete page for too long and are more likely to abandon before reading.
- **Affected metric(s):**
  - LCP: 54.3 s (mobile), 17.2 s (desktop)
  - FCP and Speed Index are also poor, reinforcing delayed visual progress.
- **Cause (most likely):**
  Heavy above-the-fold payload and delayed critical rendering (large media, script activity, and non-critical assets competing with main content).
- **Likely solution:**
  Prioritize hero content and critical resources:
  - Preload/prioritize the LCP image or headline block.
  - Delay non-critical scripts and below-the-fold media.
  - Reduce render-blocking CSS/JS and inline only minimal critical CSS.

## 2) Severe main-thread blocking delays interactivity

- **How this affects users:**
  The page may look partially loaded but feels unresponsive (scroll, tap, open menu), creating frustration and perceived slowness.
- **Affected metric(s):**
  - TBT: 18,020 ms (mobile), 4,530 ms (desktop)
  - TTI: 56.8 s (mobile), 21.0 s (desktop)
  - Max Potential FID: 8,460 ms (mobile), 1,530 ms (desktop)
- **Cause (most likely):**
  Excessive JavaScript execution and long tasks on the main thread, likely from third-party and non-critical scripts loaded too early.
- **Likely solution:**
  Reduce and defer JavaScript work:
  - Remove or delay non-essential third-party tags.
  - Split bundles and lazy-load non-critical code paths.
  - Break long tasks into smaller chunks and move work off main thread where possible.

## 3) Excess unused JavaScript increases download and execution cost

- **How this affects users:**
  Users download and parse code they do not need immediately, which slows startup and increases data usage.
- **Affected metric(s):**
  - Lighthouse opportunity: unused JavaScript savings of ~11,660 ms (mobile), ~2,060 ms (desktop)
  - Impacts TBT, TTI, and LCP.
- **Cause (most likely):**
  Over-bundled scripts and broad third-party libraries loaded on initial page view regardless of user path.
- **Likely solution:**
  Ship less code on first load:
  - Code split by route/component.
  - Use dynamic import for non-critical features.
  - Audit third-party scripts and remove low-value tags.

## 4) Poor visual progress creates a slow perceived experience

- **How this affects users:**
  Even if the server responds quickly, content fills in too slowly, so the site feels broken or unstable during load.
- **Affected metric(s):**
  - Speed Index: 22.3 s (mobile), 15.7 s (desktop)
  - FCP: 9.1 s (mobile), 3.4 s (desktop)
- **Cause (most likely):**
  Too much blocking work before meaningful paint (render path not optimized for above-the-fold text/image delivery).
- **Likely solution:**
  Improve critical rendering path:
  - Preconnect to key origins and preload critical assets.
  - Defer non-critical CSS/JS.
  - Reduce above-the-fold complexity and media weight.

## 5) Best Practices score indicates reliability/quality issues under load

- **How this affects users:**
  Runtime errors and weaker implementation quality can lead to broken features, inconsistent behavior, and trust loss.
- **Affected metric(s):**
  - Best Practices: 54 (mobile and desktop)
  - Related diagnostics include browser console errors during the run.
- **Cause (most likely):**
  Script/runtime issues and integration quality gaps (including third-party dependencies and failed resource calls).
- **Likely solution:**
  Raise implementation quality and stability:
  - Fix console errors and failed requests.
  - Validate third-party scripts and remove unstable dependencies.
  - Add monitoring for JS errors and regressions in CI.

## Priority Summary

Based on severity and user impact, the initial remediation order should be:

1. Reduce JavaScript main-thread work (TBT/TTI)
2. Improve LCP on mobile and desktop
3. Remove unused JS/CSS from initial load
4. Improve critical rendering path for FCP/Speed Index
5. Resolve best-practice/runtime reliability issues
