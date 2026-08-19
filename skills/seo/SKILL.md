---
name: seo
description: |
  Metadata, structured data, sitemaps, Core Web Vitals, crawlability, and rendering strategies for Next.js. Use when auditing or implementing technical SEO.
  Triggers: "SEO", "metadata", "sitemap", "search ranking", "page speed", "Core Web Vitals", "structured data", "Google indexing", "not showing up in search"
lastReviewed: 2026-04-10
upstreamDeps: [nextjs]
---

# SEO Specialist

Use this skill when auditing or implementing technical SEO — metadata, structured data, sitemaps, Core Web Vitals, crawlability, and rendering strategies (especially for Next.js). Focus on the technical foundations that make content discoverable, indexable, and rankable.

## Core principles

1. **Server-render for search engines** — Put critical content in the initial HTML response. Treat client-rendered content as unreliable for indexing.
2. **Structured data drives rich results** — Use JSON-LD markup to enable rich snippets, knowledge panels, and enhanced search listings.
3. **Performance is a ranking factor** — Core Web Vitals (LCP, CLS, INP) directly affect search rankings.
4. **Crawl budget matters** — Make it easy for search engines to find and index important pages. Do not waste crawl budget on duplicate, thin, or irrelevant pages.

## Technical SEO checklist

### Metadata
- Ensure every page has a unique, descriptive `<title>` (50-60 characters)
- Ensure every page has a unique `<meta name="description">` (150-160 characters)
- Include Open Graph tags (`og:title`, `og:description`, `og:image`, `og:url`) for social sharing
- Include Twitter Card tags (`twitter:card`, `twitter:title`, `twitter:description`, `twitter:image`)
- Set canonical URLs on every page to prevent duplicate content issues
- Use Next.js `metadata` export or `generateMetadata` for dynamic pages

### Structured data (JSON-LD)
- Include `Organization` schema on the homepage
- Include `BreadcrumbList` for navigation hierarchy
- Include `Article` or `BlogPosting` for content pages
- Include `Product` and `Offer` for eCommerce pages
- Include `FAQPage` for FAQ sections
- Include `LocalBusiness` for location-based businesses
- Validate with Google's Rich Results Test

### Sitemaps and robots
- Provide dynamic `sitemap.xml` generated from all indexable pages
- Ship a `robots.txt` that allows crawling of important pages and blocks irrelevant ones
- Submit the sitemap to Google Search Console
- Do not use `noindex` on pages that should be indexed
- Use `noindex` on utility pages (auth, settings, admin)

### Crawlability
- Ensure internal links use `<a href>` (not JavaScript-only navigation)
- Ensure important pages are reachable within 3 clicks from the homepage
- Avoid orphaned pages (every page has at least one internal link)
- For pagination, use `rel="next"` and `rel="prev"` or load-more patterns
- Keep redirect chains minimal (no more than one redirect hop)

### Rendering strategy
- Prefer SSG for static content pages (blog posts, landing pages, documentation)
- Prefer SSR for dynamic pages that need fresh data (dashboards, search results)
- Prefer ISR for content that changes periodically (product pages, CMS content)
- Use client-side rendering only for interactive widgets that don't need indexing

### Performance (SEO impact)
- Target LCP under 2.5 seconds
- Target CLS under 0.1
- Target INP under 200ms
- Optimize images with `next/image` (proper dimensions, lazy loading, modern formats)
- Avoid render-blocking resources
- Optimize font loading (`font-display: swap` or `optional`)

### International SEO
- Add `hreflang` tags for multi-language sites
- Use language-specific URLs (subdirectories or subdomains)
- Implement proper locale detection and redirection

## Review checklist

- Every public page has unique title and description
- Structured data is present and validates without errors
- `sitemap.xml` includes all indexable pages
- `robots.txt` is correctly configured
- Canonical URLs prevent duplicate content
- Internal linking structure is logical and complete
- Images have descriptive `alt` text
- Heading hierarchy is correct (`h1` → `h2` → `h3`)
- 404 pages return proper status codes (not soft 404s)
- Use 301 for permanent redirects, 308 for permanent with method preservation

## Common issues to flag

- Missing or duplicate page titles
- Client-rendered content that search engines can't see
- Missing structured data on key pages
- Broken internal links or redirect chains
- Missing `sitemap.xml` or outdated sitemap
- `noindex` accidentally applied to important pages
- Images without `alt` text
- Soft 404s (returning 200 for pages that don't exist)
- Missing canonical URLs on paginated or filtered pages
- Slow page load times affecting Core Web Vitals
