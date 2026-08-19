# Rendering Performance

## Browser rendering pipeline

Every frame the browser produces follows this pipeline:

```
JavaScript → Style → Layout → Paint → Composite
```

- **Style** — match CSS selectors to elements, compute final styles
- **Layout** — calculate geometry (position, size) of every element
- **Paint** — fill pixels for each layer (text, colors, borders, shadows)
- **Composite** — combine layers in GPU and draw to screen

Changing a property can trigger different parts of the pipeline:

| Trigger level | Properties | Cost |
|---------------|-----------|------|
| **Layout** (most expensive) | `width`, `height`, `padding`, `margin`, `top`, `left`, `right`, `bottom`, `font-size`, `display`, `position` | Triggers layout → paint → composite |
| **Paint** | `color`, `background`, `box-shadow`, `border-radius`, `outline`, `visibility`, `filter`, `backdrop-filter` | Triggers paint → composite |
| **Composite only** (cheapest) | `transform`, `opacity`, `will-change` | GPU only, no main thread work |

**Rule:** animate only `transform` and `opacity` whenever possible.

## Critical rendering path

The critical rendering path is the sequence of steps the browser takes to render the first meaningful paint. Shorter path = faster first paint.

### HTML optimization

- Minimize document size — remove unnecessary whitespace, comments, and inline styles in production
- Place `<link rel="stylesheet">` in `<head>` — CSS is render-blocking by default, so deliver it early
- Place `<script>` before `</body>` or use `defer` — scripts block HTML parsing unless deferred
- Avoid deep DOM nesting — deeper trees increase style calculation and layout cost
- Target < 1500 DOM nodes total; flag pages exceeding 3000

### Render-blocking resources

A resource is render-blocking if the browser must download and process it before first paint.

**CSS is render-blocking by default.** Mitigate with:

```html
<!-- Inline critical above-the-fold CSS -->
<style>/* critical styles */</style>

<!-- Defer non-critical CSS -->
<link rel="preload" href="/non-critical.css" as="style" onload="this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="/non-critical.css"></noscript>

<!-- Media-specific CSS only blocks for matching media -->
<link rel="stylesheet" href="/print.css" media="print">
```

**JavaScript is parser-blocking by default.** Mitigate with:

```html
<!-- defer: download in parallel, execute after HTML parsing, in order -->
<script src="/app.js" defer></script>

<!-- async: download in parallel, execute immediately when ready, no order guarantee -->
<script src="/analytics.js" async></script>

<!-- module: deferred by default -->
<script type="module" src="/app.js"></script>
```

When to use each:
- `defer` — app scripts that depend on DOM or other scripts
- `async` — independent scripts (analytics, ads) where execution order doesn't matter
- `type="module"` — modern ES module scripts (inherently deferred)
- Inline `<script>` — only for tiny critical bootstrapping code (< 1KB)

### Critical CSS extraction

Extract only the CSS needed for above-the-fold content and inline it:

1. Identify above-the-fold elements at common viewport sizes (1280×720, 375×667)
2. Collect all CSS rules that apply to those elements
3. Inline the result in `<style>` in `<head>`
4. Load the full stylesheet asynchronously

Tools: `critical` (npm package), Critters (webpack/vite plugin), or framework built-ins.

## CSS performance

### Selector efficiency

Browsers match selectors right-to-left. The rightmost part (key selector) matters most.

```css
/* Slow — key selector is *, browser checks every element */
.sidebar > * { }

/* Slow — key selector is div, very common element */
.header .nav .list .item div { }

/* Fast — key selector is a class, specific match */
.nav-item { }

/* Fast — ID selectors are the fastest */
#main-nav { }
```

Guidelines:
- Avoid universal selector (`*`) as key selector
- Keep selector depth ≤ 3 levels
- Prefer class selectors over tag or attribute selectors
- Avoid `@import` in CSS — it serializes downloads; use `<link>` tags instead

### Style recalculation

Style recalculation cost scales with the number of elements and the complexity of selectors. Reduce cost by:

- Keeping selectors simple (class-based, shallow)
- Reducing total DOM size
- Avoiding broad invalidation (changing a class on `<body>` recalculates everything)
- Using BEM or similar conventions where selector specificity is flat

### CSS containment

`contain` tells the browser that an element's internals are independent from the rest of the page, allowing rendering optimizations.

```css
/* Layout containment — element's layout doesn't affect/depend on siblings */
.card { contain: layout; }

/* Size containment — element's size is independent of children */
.card { contain: size; }

/* Paint containment — nothing inside paints outside this element's bounds */
.card { contain: paint; }

/* Style containment — counters/quotes scoped to this subtree */
.card { contain: style; }

/* Shorthand for layout + paint + style (the most common useful combination) */
.card { contain: content; }

/* All containment types (requires explicit size since size containment means children don't affect it) */
.widget { contain: strict; width: 300px; height: 200px; }
```

### content-visibility

`content-visibility: auto` skips rendering of off-screen elements entirely — no layout, paint, or style calculation until they scroll near the viewport.

```css
.long-section {
  content-visibility: auto;
  /* REQUIRED: explicit height estimate to prevent scrollbar jumping */
  contain-intrinsic-size: auto 500px;
}
```

Use on:
- Long lists or feeds
- Below-the-fold page sections
- Tab panels that aren't visible
- Accordion content that's collapsed

Do NOT use on:
- Above-the-fold content (delays first paint)
- Elements with `position: fixed` or `position: sticky`
- Elements that contain scroll-linked animations

### will-change

`will-change` promotes an element to its own compositor layer before animation starts.

```css
/* Apply right before the animation starts */
.card:hover { will-change: transform; }
.card.animating { will-change: transform, opacity; }
```

Rules:
- **Never apply permanently** — `will-change: transform` on every element wastes GPU memory
- **Apply just before the change** — use `:hover`, a class toggle, or `requestAnimationFrame`
- **Remove after animation completes** — reset to `will-change: auto`
- **Don't use as a performance hack** — it creates a new layer; too many layers hurt compositing

## Layout performance

### Forced synchronous layout (layout thrashing)

Reading layout properties after writing styles forces the browser to calculate layout immediately instead of batching.

```javascript
// BAD — triggers layout on every iteration
for (const el of elements) {
  const width = el.offsetWidth    // read → forces layout
  el.style.width = width + 10 + "px" // write → invalidates layout
}

// GOOD — batch reads, then batch writes
const widths = []
for (const el of elements) {
  widths.push(el.offsetWidth) // all reads first
}
for (let i = 0; i < elements.length; i++) {
  elements[i].style.width = widths[i] + 10 + "px" // all writes after
}
```

Layout-triggering properties (reading any of these forces layout if styles are dirty):
`offsetTop/Left/Width/Height`, `scrollTop/Left/Width/Height`, `clientTop/Left/Width/Height`, `getComputedStyle()`, `getBoundingClientRect()`, `innerText` (computes layout to determine visibility)

### Reducing layout scope

- Use `contain: layout` on components that change independently
- Avoid changing layout properties on elements high in the DOM tree (affects more children)
- Use `transform` for position changes instead of `top`/`left`
- Use `flexbox` or `grid` instead of JavaScript-based layouts

### DOM size

| DOM nodes | Impact |
|-----------|--------|
| < 1500 | Good |
| 1500–3000 | Monitor closely |
| > 3000 | Likely causing layout/paint performance issues |

Reduce DOM size by:
- Virtualizing long lists (only render visible items + buffer)
- Using `content-visibility: auto` for off-screen sections
- Lazy-loading component subtrees (render on demand, not on mount)
- Removing wrapper `<div>` elements that serve no purpose

## Compositing and GPU layers

Each compositor layer is a texture uploaded to the GPU. Layers are cheap to move/transform but expensive to create and consume VRAM.

### What creates a new layer

- `transform: translateZ(0)` or `translate3d(0, 0, 0)` (hack — prefer `will-change`)
- `will-change: transform | opacity | filter`
- `position: fixed`
- `<video>`, `<canvas>`, `<iframe>` elements
- Elements with CSS `filter` or `backdrop-filter`
- Overlapping elements near a composited layer (implicit promotion)

### Layer management

- Use Chrome DevTools "Layers" panel to audit layer count and memory
- Avoid promoting too many elements — each layer costs ~1-4MB VRAM
- Watch for implicit layer promotion (elements overlapping a composited layer get promoted too)
- Remove `will-change` after animations complete

## Animation performance

### High-performance animations

```css
/* GOOD — composite-only properties, 60fps */
.element {
  transition: transform 300ms ease, opacity 300ms ease;
}

/* BAD — triggers layout on every frame */
.element {
  transition: width 300ms ease, top 300ms ease;
}
```

### Animation checklist

- Only `transform` and `opacity` (composite-only)
- Duration ≤ 300ms for UI feedback, ≤ 500ms for transitions
- Use `prefers-reduced-motion` to disable or reduce animations
- Avoid animating during page load (delays time-to-interactive)
- Use CSS animations/transitions over JavaScript when possible (run on compositor thread)
- For complex sequences, use Web Animations API or `requestAnimationFrame`

### prefers-reduced-motion

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

## Common rendering issues to flag

- CSS animations on `width`, `height`, `top`, `left`, `margin`, or `padding`
- `will-change` applied permanently in stylesheets instead of dynamically
- Layout thrashing in JavaScript (interleaved reads and writes)
- Deep DOM nesting (> 30 levels) or large DOM (> 3000 nodes)
- Synchronous `<script>` in `<head>` without `defer`/`async`
- `@import` in CSS files (serializes downloads)
- Missing `content-visibility: auto` on long off-screen sections
- Overly complex CSS selectors (deep nesting, universal key selectors)
- Missing `contain` on independently updating components
- Render-blocking CSS that includes styles for below-the-fold content
