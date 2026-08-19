---
name: web-performance
description: |
  Core Web Vitals optimization, resource loading, image and font optimization, caching, bundle size reduction, CSS and rendering performance, runtime JavaScript performance, and monitoring. Use when optimizing page speed, fixing performance regressions, or implementing performance best practices.
  Triggers: "optimize performance", "page is slow", "improve load time", "Core Web Vitals", "reduce bundle size", "lazy load", "caching strategy", "font optimization", "resource hints", "web performance", "layout shift", "LCP", "CLS", "INP", "render blocking"
lastReviewed: 2026-04-13
upstreamDeps: []
---

# Web Performance

Use this skill when optimizing web performance. Framework-agnostic patterns — for Next.js-specific guidance, defer to the web-performance-auditor agent.

## How this skill is organized

SKILL.md is a routing document. Deep guidance lives in reference files — read only what the task requires.

| Reference file | When to read it |
|----------------|-----------------|
| [rendering.md](rendering.md) | Slow first paint, render-blocking resources, layout thrashing, CSS containment, compositing, GPU layers |
| [javascript.md](javascript.md) | Large bundles, slow interactions, memory leaks, long tasks, code splitting, Web Workers |
| [assets.md](assets.md) | Image optimization, font loading, responsive images, AVIF/WebP, font subsetting |
| [network.md](network.md) | Caching, CDN, compression, resource hints, service workers, third-party script impact |
| [core-web-vitals.md](core-web-vitals.md) | Diagnosing and fixing LCP, CLS, or INP failures with step-by-step playbooks |
| [monitoring.md](monitoring.md) | Setting up RUM, lab testing, performance budgets, CI gates, alerting |

## Core principles

1. **Measure before optimizing** — profile with DevTools, Lighthouse, or WebPageTest before changing anything. Guessing at bottlenecks wastes effort.
2. **Critical path first** — optimize what blocks rendering: HTML, CSS, fonts, above-the-fold images. Everything else can wait.
3. **Less is more** — the fastest code is code you don't ship. Remove unused dependencies, dead code, and unnecessary polyfills before reaching for compression tricks.
4. **Budget everything** — set concrete budgets for bundle size, LCP, CLS, and INP. Enforce them in CI.

## Core Web Vitals targets

| Metric | Good | Needs improvement | Poor |
|--------|------|-------------------|------|
| **LCP** | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **CLS** | ≤ 0.1 | ≤ 0.25 | > 0.25 |
| **INP** | ≤ 200ms | ≤ 500ms | > 500ms |

## Quick diagnosis guide

**Page loads slowly (high LCP)**
→ Read [core-web-vitals.md](core-web-vitals.md) (LCP section), then [rendering.md](rendering.md) for render-blocking issues, [assets.md](assets.md) for image/font problems, [network.md](network.md) for caching gaps.

**Content jumps around (high CLS)**
→ Read [core-web-vitals.md](core-web-vitals.md) (CLS section), then [assets.md](assets.md) for image/font dimensions, [rendering.md](rendering.md) for dynamic content injection patterns.

**Interactions feel sluggish (high INP)**
→ Read [core-web-vitals.md](core-web-vitals.md) (INP section), then [javascript.md](javascript.md) for long tasks, main thread optimization, and scheduling.

**Bundle too large**
→ Read [javascript.md](javascript.md) (Bundle optimization section).

**Third-party scripts dragging performance**
→ Read [network.md](network.md) (Third-party scripts section).

**Need to set up performance tracking**
→ Read [monitoring.md](monitoring.md).

## Review checklist

- [ ] LCP element preloaded, not lazy-loaded, has `fetchpriority="high"`
- [ ] All images have explicit `width` and `height`
- [ ] Fonts self-hosted, subsetted, preloaded, `font-display` set
- [ ] No render-blocking scripts in `<head>` without `defer`/`async`
- [ ] Bundle within budget, unused deps removed, code-split by route
- [ ] Third-party scripts loaded async, heavy embeds behind facades
- [ ] HTTP cache headers correct for each resource type
- [ ] No long tasks (> 50ms) on interaction-critical paths
- [ ] `content-visibility: auto` on long off-screen sections
- [ ] CSS animations use only `transform` and `opacity`
- [ ] Real-user monitoring in place for Core Web Vitals
