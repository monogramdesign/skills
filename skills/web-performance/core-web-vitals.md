# Core Web Vitals: Diagnostic Playbooks

## LCP (Largest Contentful Paint)

**Target: ≤ 2.5s**

LCP measures when the largest visible content element finishes rendering. Usually a hero image, heading, or large text block.

### Diagnostic steps

1. **Identify the LCP element**: Chrome DevTools → Performance panel → Timings track → click LCP marker. Or Lighthouse → "Largest Contentful Paint element."
2. **Break down the time**: LCP = TTFB + Resource Load Delay + Resource Load Duration + Element Render Delay.
3. **Find which sub-part is slow** and follow the appropriate fix below.

### Sub-part 1: Slow TTFB (Time to First Byte)

The server takes too long to respond.

**Diagnosis:** DevTools Network panel → select the HTML document → check "Waiting (TTFB)." As a diagnostic heuristic, aim for < 800ms (not a CWV threshold, but slow TTFB directly eats into LCP budget).

**Fixes (in priority order):**

| Fix | Details |
|-----|---------|
| Use a CDN | Serve HTML from the edge, not a distant origin |
| Enable HTTP caching for HTML | `Cache-Control: public, s-maxage=60, stale-while-revalidate=300` |
| Use 103 Early Hints | CDN sends preload hints while origin is still processing |
| Optimize server-side rendering | Profile backend; database queries, API calls, template rendering |
| Eliminate redirect chains | Each redirect adds a full roundtrip; redirect at the CDN instead |
| Use streaming SSR | Send HTML in chunks as data becomes available |

### Sub-part 2: Resource load delay

The LCP resource (image, font) starts downloading too late.

**Diagnosis:** DevTools Network panel → find the LCP resource → check the "Stalled" and "Waiting" columns. Look at the waterfall to see when the request starts relative to page load.

**Fixes:**

| Fix | Details |
|-----|---------|
| Preload the LCP resource | `<link rel="preload" href="..." as="image" fetchpriority="high">` |
| Use `<img>` not CSS `background-image` | The preload scanner can discover `<img>` immediately |
| Add `fetchpriority="high"` | Tells the browser to prioritize this resource |
| Preconnect to the image origin | `<link rel="preconnect" href="https://cdn.example.com">` |
| Don't lazy-load the LCP image | `loading="lazy"` delays loading until near-viewport |
| Inline the CSS that reveals the LCP element | Don't hide it behind an external stylesheet download |

### Sub-part 3: Resource load duration

The LCP resource takes too long to download.

**Diagnosis:** DevTools Network panel → check "Content Download" time for the LCP resource.

**Fixes:**

| Fix | Details |
|-----|---------|
| Compress images | AVIF/WebP instead of JPEG/PNG |
| Right-size images | Serve the size needed for the viewport, not the largest version |
| Use an image CDN | Automatic format conversion and resizing |
| Enable Brotli compression | For text-based LCP elements (large headings) |
| Reduce server response size | Remove unnecessary response headers, compress |

### Sub-part 4: Element render delay

The LCP element is loaded but not yet rendered.

**Diagnosis:** The resource has finished downloading but LCP hasn't fired yet. Usually means render-blocking CSS or JavaScript is delaying layout/paint.

**Fixes:**

| Fix | Details |
|-----|---------|
| Remove render-blocking CSS | Inline critical CSS, defer the rest |
| Remove render-blocking JavaScript | Use `defer` or `async` |
| Avoid `display: none` then reveal | Element must be visible to count as LCP |
| Avoid JavaScript-driven rendering for LCP content | Server-render the LCP element in HTML |
| Minimize CSS size | Smaller CSS = faster parse = faster render |

---

## CLS (Cumulative Layout Shift)

**Target: ≤ 0.1**

CLS measures visual stability — how much visible content shifts unexpectedly during the page's lifetime.

### Diagnostic steps

1. **Find layout shift sources**: Chrome DevTools → Performance panel → Experience track → click "Layout Shift" entries. The shifted elements are highlighted.
2. **Check which elements shifted** and what caused them to move.
3. **Use the `layout-shift` PerformanceObserver** in production to identify real-user shift sources.

```javascript
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      for (const source of entry.sources) {
        console.log("Shift source:", source.node, "by", entry.value)
      }
    }
  }
}).observe({ type: "layout-shift", buffered: true })
```

### Common CLS causes and fixes

**Images without dimensions**

```html
<!-- BAD — browser doesn't know image size until it loads -->
<img src="/photo.jpg" alt="...">

<!-- GOOD — browser reserves space immediately -->
<img src="/photo.jpg" alt="..." width="800" height="600">

<!-- GOOD — aspect-ratio in CSS -->
<img src="/photo.jpg" alt="..." style="aspect-ratio: 4/3; width: 100%;">
```

**Dynamic content injected above existing content**

- Banners, cookie notices, ads injected at the top of the page
- Fix: reserve space with `min-height` or use `position: fixed`/`sticky` overlays instead of inserting into document flow

**Web fonts causing text reflow (FOUT)**

When the web font loads, text re-renders with different metrics, shifting content.

| Fix | How |
|-----|-----|
| `font-display: optional` | Uses fallback permanently if font loads late (zero CLS) |
| Adjust fallback metrics | `size-adjust`, `ascent-override`, `descent-override` on fallback `@font-face` |
| Preload fonts | `<link rel="preload" as="font" crossorigin>` reduces swap likelihood |

**Late-loading content that pushes things down**

- Async API data that inserts content above or between existing elements
- Fix: use skeleton placeholders with the exact dimensions of the final content
- Fix: load data before rendering the page (SSR or data loaders)

**CSS animations on layout properties**

```css
/* BAD — animating height shifts siblings */
.accordion { transition: height 300ms; }

/* BETTER — max-height avoids explicit height calculation but still triggers layout */
.accordion { max-height: 0; overflow: hidden; transition: max-height 300ms; }

/* GOOD — grid row transition avoids fixed height values */
.accordion { display: grid; grid-template-rows: 0fr; transition: grid-template-rows 300ms; }
.accordion.open { grid-template-rows: 1fr; }
.accordion > div { overflow: hidden; }
```

**Dynamic viewport units on mobile**

- `100vh` changes when the mobile browser chrome shows/hides
- Fix: use `100dvh` (dynamic viewport height) or `100svh` (small viewport height)

**Ads and embeds without reserved space**

```css
/* Reserve space for ad slots */
.ad-slot {
  min-height: 250px;
  min-width: 300px;
}
```

### CLS from scrolling (not counted but still bad UX)

CLS only counts shifts without recent user input. But scroll-triggered shifts (e.g., sticky headers resizing, infinite scroll changing layout) still hurt UX. Fix them with the same techniques.

---

## INP (Interaction to Next Paint)

**Target: ≤ 200ms**

INP measures the latency from user interaction (click, tap, keypress) to the next visual update. It captures the worst interaction latency over the page's lifetime (with outliers discounted).

### INP breakdown

```
INP = Input Delay + Processing Time + Presentation Delay
```

- **Input delay** — time between the physical interaction and the event handler starting (caused by main thread being busy)
- **Processing time** — time spent in the event handler(s)
- **Presentation delay** — time from handler completion to the browser painting the visual update

### Diagnostic steps

1. **Identify slow interactions**: Chrome DevTools → Performance panel → Interactions track. Look for interactions with total duration > 200ms.
2. **Check Input Delay**: was the main thread busy when the user interacted? Look for long tasks immediately before the interaction.
3. **Check Processing Time**: how long did the event handler take? Look for the handler's flame chart.
4. **Check Presentation Delay**: was there a long style/layout/paint after the handler?

### Use PerformanceObserver in production

```javascript
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 200) {
      console.log("Slow interaction:", {
        type: entry.name,
        duration: entry.duration,
        inputDelay: entry.processingStart - entry.startTime,
        processingTime: entry.processingEnd - entry.processingStart,
        presentationDelay: entry.startTime + entry.duration - entry.processingEnd,
      })
    }
  }
}).observe({ type: "event", durationThreshold: 16, buffered: true })
```

### High input delay fixes

The main thread was busy when the user tried to interact.

| Fix | Details |
|-----|---------|
| Break up long tasks | Use `scheduler.yield()`, `setTimeout(0)`, or chunk processing |
| Defer non-critical JS | Move analytics, chat widgets, A/B testing to `requestIdleCallback` or `defer` |
| Reduce JavaScript execution on load | Code-split, lazy-load, remove unused JS |
| Use Web Workers | Move CPU-intensive work off main thread |
| Eliminate long-running `useEffect`/`componentDidMount` | Move data fetching to server, reduce client-side work |

### High processing time fixes

The event handler itself is slow.

| Fix | Details |
|-----|---------|
| Simplify event handlers | Do minimal work in the handler; defer heavy computation |
| Avoid synchronous layout | Don't read layout properties in event handlers (triggers forced layout) |
| Debounce repeat interactions | Input, scroll, resize handlers should be debounced/throttled |
| Batch state updates | In React: state updates in event handlers are already batched; avoid `flushSync` |
| Use `startTransition` | In React: mark non-urgent updates as transitions |
| Move computation to a Worker | Heavy data processing should not run in event handlers |

### High presentation delay fixes

The browser takes too long to render the visual update after the handler completes.

| Fix | Details |
|-----|---------|
| Reduce DOM size | Fewer nodes = faster style/layout/paint |
| Use `content-visibility: auto` | Skip rendering off-screen content |
| Avoid complex CSS selectors | Faster style recalculation |
| Reduce paint complexity | Avoid `box-shadow`, `filter`, `blur` on large areas |
| Use CSS `contain` | Limit the scope of layout/paint recalculation |
| Minimize DOM mutations | Batch changes, use `requestAnimationFrame` for visual updates |

### Quick INP wins

1. Add `passive: true` to all `touchstart`, `touchmove`, `wheel` listeners that don't call `preventDefault()`
2. Use `requestAnimationFrame` to defer visual updates out of event handlers
3. Replace `mouseenter`/`mouseleave` heavy handlers with CSS `:hover`
4. Use CSS transitions instead of JavaScript animation in response to interactions
5. Yield to the main thread between expensive steps in click handlers

## Field measurement

For `web-vitals` library setup, RUM platforms, and CI budget enforcement, see [monitoring.md](monitoring.md).

## Common Core Web Vitals issues to flag

- LCP image loaded without `preload` or `fetchpriority="high"`
- LCP image is lazy-loaded
- Render-blocking CSS/JS delaying LCP
- Images without `width`/`height` causing CLS
- Font swap without adjusted fallback metrics causing CLS
- Dynamic content insertion above the fold causing CLS
- Long tasks (> 50ms) on the main thread degrading INP
- Heavy event handlers without debouncing
- Forced synchronous layout in interaction handlers
- Missing `passive: true` on touch/wheel listeners
- No field measurement for Core Web Vitals in production
