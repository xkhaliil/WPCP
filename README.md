# Course Project Performance Audit Report

## Target Website

- **Name:** AP News
- **URL:** https://apnews.com/

## Why AP News Is a Good Audit Candidate

AP News is a high-traffic, content-heavy news platform with a mix of article pages, hubs, media-heavy pages, and dynamic features such as search and ads. This makes it a strong candidate for a performance audit because:

- It represents real-world complexity (third-party scripts, ads, media, and frequent content updates).
- It includes different page templates that likely perform differently under load.
- Improvements here would have clear user impact on reading speed, engagement, and accessibility.

## Main PageSpeed Insights Scores (Fresh Lighthouse Run)

Homepage tested: https://apnews.com/  
Measurement date: 2026-07-06  
Method: Lighthouse (mobile default + desktop preset)

### Desktop

- **Performance:** 25
- **Accessibility:** 79
- **Best Practices:** 54
- **SEO:** 85

**Performance details**

- **First Contentful Paint (FCP):** 3.4 s
- **Largest Contentful Paint (LCP):** 17.2 s
- **Total Blocking Time (TBT):** 4,530 ms
- **Cumulative Layout Shift (CLS):** 0.064
- **Speed Index:** 15.7 s

### Mobile

- **Performance:** 25
- **Accessibility:** 76
- **Best Practices:** 54
- **SEO:** 85

**Performance details**

- **First Contentful Paint (FCP):** 9.1 s
- **Largest Contentful Paint (LCP):** 54.3 s
- **Total Blocking Time (TBT):** 18,020 ms
- **Cumulative Layout Shift (CLS):** 0.033
- **Speed Index:** 22.3 s

## Audit Focus Pages (8)

1. **https://apnews.com/**  
   Main landing page with top stories, ads, and mixed content blocks. Critical for first impressions and overall site performance baseline.

2. **https://apnews.com/entertainment**  
   Section hub page likely using a category template with many thumbnails and feed items. Useful for comparing section-page rendering against homepage.

3. **https://apnews.com/hub/fifa-world-cup**  
   Topic hub page with aggregated content. Important for evaluating repeated card layouts, pagination/infinite feeds, and caching behavior.

4. **https://apnews.com/photo-gallery/world-cup-photos-mbappe-haaland-jimenez-57fd3b1070ed79152dfa89b7319f6139**  
   Media-heavy gallery page. Key for auditing image optimization, lazy loading, responsive image behavior, and visual stability.

5. **https://apnews.com/article/world-cup-schedule-results-news-81645977a722c4020c9644d17589bdbb**  
   Typical long-form article page. Essential for measuring real reading experience, ad/script impact, and Core Web Vitals on article templates.

6. **https://apnews.com/hub/quizzes**  
   Interactive content hub. Useful for testing JavaScript execution costs, interactivity delays, and Total Blocking Time patterns.

7. **https://apnews.com/search?q=world+cup**  
   Search results page with dynamic query handling. Important for measuring backend/API latency effects and client-side rendering overhead.

8. **https://apnews.com/donate**  
   Conversion-oriented page where speed directly affects user completion rates. Valuable for evaluating performance impact on business/goal outcomes.

## Assignment Artifacts

- Baseline report: [baseline.md](baseline.md)
- Findings report: [findings.md](findings.md)
