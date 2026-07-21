# AP News — Performance Audit

## Executive Summary

**Audit date:** July 2026  
**Audited page:** https://apnews.com/  
**Report audience:** Directors, product owners, budget holders

---

## The Bottom Line

AP News delivers its stories to the server in 40 milliseconds. Then it makes readers wait up to 54 seconds before they can read anything.

That gap — between a fast server and a slow reader experience — is entirely self-inflicted. It is caused by the volume and loading order of third-party advertising scripts, a single oversized JavaScript bundle, and a stylesheet that blocks the browser from rendering any content until it has been fully downloaded. None of these problems require new infrastructure. All of them are fixable in a normal engineering sprint cycle.

Google's public measurement tool scores AP News at **25 out of 100** on performance — the bottom quartile of the web. The industry standard for news sites is 45–55. Competitors scoring above 50 receive a ranking advantage in search results for every query where page experience is a tiebreaker.

The audit identified six areas of concern, ranked below by business impact. It also identified three areas where AP News is already doing the right thing, which are noted at the end.

You can verify any number in this report yourself at **pagespeed.web.dev** — enter the AP News URL and run the public test.

---

## How Slow Is It, in Terms That Matter

| Measure                                        | AP News today | Google's "good" threshold | Gap             |
| ---------------------------------------------- | ------------- | ------------------------- | --------------- |
| Time until the main story is visible (desktop) | 17.2 seconds  | 2.5 seconds               | 7× over         |
| Time until the main story is visible (mobile)  | 54.3 seconds  | 2.5 seconds               | 22× over        |
| Time until a reader can tap or scroll (mobile) | 56.8 seconds  | —                         | —               |
| Google Performance score                       | 25 / 100      | 90 / 100                  | Bottom quartile |

Google's own research shows that **53% of mobile visitors leave a page that takes longer than 3 seconds to load.** AP News's mobile load time is 54 seconds under standard mobile network conditions. The majority of mobile visitors are gone before the headline appears.

A one-second improvement in page load time is associated with an 11% increase in pageviews and a 7% increase in conversions (Google / Deloitte, 2019). The gap here is not one second — it is tens of seconds. Even closing half of it would represent a measurable audience and revenue impact.

---

## Ranked Opportunities

The six findings below are sorted by the ratio of business impact to engineering effort. The top two are quick wins. The fourth is structural and more expensive, but addresses the root cause of the worst numbers.

---

### 1. Clear the rendering path — all pages, all readers, fast win

**The problem:** The server responds in 40 milliseconds. The browser then waits another 4 seconds before it can show a single pixel. It is blocked by a stylesheet and four advertising/analytics scripts that are placed in the page's loading sequence before any content. This is not a server problem. It is a loading-order problem.

**Business impact:** Affects every visitor on every page. Fixing this alone could bring the time-to-first-content from 3.4 seconds (desktop) to under 1 second, and from 9.1 seconds (mobile) toward 2–3 seconds. That is the difference between meeting Google's page experience threshold and failing it — with direct consequences for search ranking.

**Cost to fix:** One engineering sprint. The work involves reordering the stylesheet to load asynchronously after a small set of critical styles is inlined, and marking four blocking scripts as non-blocking. No infrastructure changes. No third-party contracts. No redesign.

**Risk of not fixing:** Every day this is unfixed, every new visitor to every page experiences a blank screen for 3–9 seconds before content appears. That is the first impression AP News makes.

---

### 2. The ad-tech stack is working against itself

**The problem:** Third-party advertising and marketing scripts account for **68% of the page's total weight** and occupy the browser's processor for nearly **10 full seconds** on desktop (20+ seconds on mobile). This includes scripts from Wunderkind (behavioral popups), Primis (video ads), Nativo, Google Tag Manager, DoubleClick, Quantcast, and 35 other vendors — all loaded simultaneously at page start.

An ad that loads slowly is an ad that does not get seen. Industry ad viewability standards count an ad as "viewable" only if 50% of it is visible for one second. When the page takes 17 seconds to become fully interactive, a meaningful portion of the ad inventory below the fold never qualifies as viewable, and programmatic buyers pay less for it.

The scripts meant to generate revenue are reducing the revenue-generating capacity of the page.

**Business impact:** Faster ad loading → higher viewability scores → higher CPM rates from programmatic buyers. The ad-tech vendors themselves benefit from a faster host page. This is not a choice between performance and ad revenue; it is a choice between ad revenue now and more ad revenue with modest engineering investment.

**Cost to fix:** One to two sprints. Lazy-load ad slots that are below the visible screen area. Replace the Primis video ad player (which downloads 3.2 MB of video code on every page load, 91% of which is immediately unused) with a lightweight placeholder that loads the full player only when it scrolls into view. This one change alone removes 3.2 MB from the initial page load.

**Risk of not fixing:** Ad revenue continues to be suppressed by the weight and sequencing of the ad-tech itself. The `html-load.cc` domain — an unrecognized vendor loading 141 KB of code with no identifiable publisher or category — should be investigated immediately. Unauthorized third-party code is a security and compliance risk in addition to a performance cost.

---

### 3. Hero image does not load first

**The problem:** The main story image — the largest visual element readers see when they arrive — loads in parallel with every other image on the page rather than first. On pages where the top story uses a video instead of a still image, the problem is compounded: a video player loads in place of a photograph, which is far heavier and takes longer to appear.

**Business impact:** The main story image is the visual hook that keeps a reader engaged. Delayed hero imagery is a direct contributor to the 17-second LCP metric. It is also the single largest factor Google's algorithm uses to assess "did the page load something useful."

**Cost to fix:** One day of engineering time. A single HTML attribute (`fetchpriority="high"`) on the hero image tells the browser to prioritize it over all other images. Using a static photograph for the top story and replacing it with the video on user click eliminates the video-player weight from the initial load.

**Risk of not fixing:** The page's most important visual element routinely loses the loading race to secondary and tertiary images. This is a disproportionately expensive problem for its simplicity to fix.

---

### 4. The JavaScript bundle delivers code for every page type on every page

**The problem:** AP News ships a single JavaScript file containing the code for the homepage, article pages, photo galleries, quiz pages, and every other page template — all at once, on every page. When a reader opens a news article, they download and run gallery-slide code, quiz logic, and homepage feed renderers that will never be used. At the homepage alone, **78.5% of this bundle is unused**.

This bundle is the second-largest contributor to the page's blocked state after initial load. Running it costs 405 milliseconds of processor time on a fast desktop; on a mid-range mobile phone, the equivalent cost triples.

**Business impact:** Every page type pays the performance cost of every other page type. An article reader on a mobile device waits through the full execution of code they will never use before they can interact with the page. This is a structural inefficiency that compounds every other performance problem — it means fixes to rendering speed and ad loading will deliver less improvement than they would in a properly split codebase.

**Cost to fix:** Two to three engineering sprints. The JavaScript build system needs to be configured to split the bundle by page type, so each route delivers only the code it needs. This is a standard practice in modern web development and is well-supported by existing tooling. It is the most expensive item on this list but also the one with the most durable long-term impact.

**Risk of not fixing:** The current architecture places a ceiling on how much the other fixes can deliver. Quick wins in items 1–3 will show measurable improvement. Without addressing the bundle architecture, the page will not reach competitive performance benchmarks regardless of other optimizations.

---

### 5. Return visitors do not benefit from caching

**The problem:** When a reader returns to AP News after their first visit, they re-download 99% of the same bytes they downloaded the first time. The warm-load transfer size is 9.45 MB — only 1% less than the 9.55 MB cold load. A news site's loyal readers — the audience with the highest lifetime value — receive no speed benefit from having visited before.

**Business impact:** Subscribers, newsletter recipients, and direct-traffic readers are the most commercially valuable audience segment. They visit frequently. Each visit costs them and the AP News infrastructure the same bandwidth as a first-time visit. Improving caching for stable assets (JavaScript, CSS, fonts, UI images) would make every return visit meaningfully faster at no marginal server cost.

**Cost to fix:** One sprint. Extend cache lifetimes on static assets with content-hash filenames (the infrastructure for this already exists — assets are already named with hashes, the cache duration just needs to be extended to one year). Apply `immutable` cache directives to these files.

**Risk of not fixing:** Loyal readers experience no performance reward for loyalty. On mobile, repeated downloads of 9+ MB per visit contribute to data costs that are visible to readers on limited data plans.

---

### 6. Accessibility gaps create legal exposure and exclude readers

**The problem:** The Lighthouse accessibility audit scores AP News at 76/100. Specific gaps include: interactive controls (buttons, navigation links) with no accessible name, images without descriptive text, and heading structure that skips levels. These gaps mean that readers using screen readers — assistive technology for visual impairment — cannot reliably navigate or use the site.

**Business impact:** In most jurisdictions, public-facing websites are expected to meet accessibility standards (WCAG 2.1 AA in the EU, ADA in the US for organizations of this size). Non-compliance is an increasingly common basis for litigation. Beyond legal risk, visually impaired readers represent a real audience that is currently excluded from AP News content.

**Cost to fix:** One to two sprints for the Lighthouse-flagged items. Adding `aria-label` attributes to icon-only buttons, providing `alt` text on images, and correcting heading order are well-understood, low-risk code changes.

**Risk of not fixing:** Legal exposure increases as accessibility litigation becomes more common. The audience that cannot use the site today cannot be converted to subscribers.

---

## What Is Already Working

Three areas are genuinely well-executed and should be maintained:

**1. Server response time is excellent.** The AP News server returns a fully-rendered HTML document in 40 milliseconds. This is faster than most news sites and demonstrates that the backend infrastructure is not the bottleneck. The performance problem is entirely in how the browser is asked to process what the server sends, not in the server itself.

**2. Compression is consistently applied.** All text-based files (HTML, JavaScript, CSS) are compressed using Brotli or gzip, achieving a 63% reduction in transfer size. Binary files such as images are not double-compressed (which would add overhead without benefit). This is correct practice and reduces bandwidth costs.

**3. The image delivery infrastructure is capable.** AP News uses a professional image CDN (DIMS) that can resize, crop, and convert images on the fly for any device size. The capability to deliver optimally sized images to every reader is already in place. The remaining gap — images delivered slightly larger than their display size — is a configuration issue, not an infrastructure one.

---

## The Investment Framing

| Finding                        | Effort      | Who benefits                    | When you see it          |
| ------------------------------ | ----------- | ------------------------------- | ------------------------ |
| 1. Clear the rendering path    | 1 sprint    | All visitors, all pages         | Immediately after deploy |
| 2. Discipline ad-tech loading  | 1–2 sprints | All visitors; ad revenue        | Within 2–4 weeks         |
| 3. Hero image priority         | 1 day       | All visitors, homepage + hubs   | Same day                 |
| 4. JavaScript bundle splitting | 2–3 sprints | All visitors; long-term ceiling | After rollout            |
| 5. Return-visit caching        | 1 sprint    | Subscribers, loyal readers      | Immediately after deploy |
| 6. Accessibility fixes         | 1–2 sprints | Screen-reader users             | After rollout            |

Items 1, 3, and 5 can be delivered in a single focused sprint and would produce measurable improvements in Google's public performance score, search ranking signals, and reader-facing load times within weeks of deployment.

Items 2 and 4 are larger in scope but address the structural causes. Without them, the page will improve but not reach competitive benchmarks.

Item 6 addresses legal and inclusion risk independently of performance.

---

_Full technical findings, root-cause analysis, and implementation specifications are available in the accompanying technical report._
