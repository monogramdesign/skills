# JavaScript Performance

## Bundle optimization

### Analysis

Always analyze before optimizing. Tools:
- **source-map-explorer** — treemap visualization from source maps
- **webpack-bundle-analyzer** / **rollup-plugin-visualizer** — interactive bundle visualization
- **bundlephobia.com** — check package size before installing
- **`import-cost`** VS Code extension — inline size display on imports

### Tree shaking

Tree shaking removes unused exports. It only works with ES modules (`import`/`export`).

```javascript
// GOOD — tree-shakeable, only debounce is bundled
import { debounce } from "lodash-es"

// BAD — entire lodash library is bundled
import _ from "lodash"

// BAD — CommonJS, not tree-shakeable
const { debounce } = require("lodash")
```

Requirements for effective tree shaking:
- Use ES module builds of libraries (`lodash-es`, not `lodash`)
- Set `"sideEffects": false` in `package.json` (or list files with side effects)
- Avoid re-exporting everything through barrel files (`index.ts` that re-exports)
- Avoid assignments to module-scoped variables (considered side effects)

### Barrel file problem

```typescript
// lib/index.ts — barrel file re-exporting everything
export { Button } from "./button"
export { Modal } from "./modal"
export { Chart } from "./chart" // Chart pulls in d3 (500KB)

// consumer.ts
import { Button } from "./lib" // May pull in Chart → d3 depending on bundler
```

Fix: import directly from the source file, not through barrel files:

```typescript
import { Button } from "./lib/button"
```

### Code splitting

Split by route and by feature so users only download what they need.

**Route-based splitting** — each route is a separate chunk, loaded on navigation:

```javascript
// Dynamic import — creates a split point
const Settings = () => import("./pages/settings")
```

**Feature-based splitting** — heavy features loaded on demand:

```javascript
// Load chart library only when the user opens the chart panel
async function showChart(data) {
  const { Chart } = await import("./chart")
  const chart = new Chart(data)
  chart.render()
}
```

**Shared chunks** — extract common dependencies into shared chunks to avoid duplication:

```javascript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
        },
      },
    },
  },
})
```

### Small alternatives

Replace heavy libraries with lighter alternatives or native APIs:

| Heavy library | Lighter alternative | Native API |
|---------------|-------------------|------------|
| `moment` (300KB) | `date-fns` (tree-shakeable) | `Intl.DateTimeFormat`, `Temporal` |
| `lodash` (70KB) | `lodash-es` (tree-shakeable) | Array/Object methods |
| `axios` (14KB) | — | `fetch` |
| `classnames` (1KB) | — | Template literals |
| `uuid` (4KB) | — | `crypto.randomUUID()` |
| `chalk` (node, 40KB) | `picocolors` (2KB) | — |

### Dead code elimination

- Remove unused npm dependencies (`depcheck` or `knip`)
- Remove unused exports (`ts-prune` or `knip`)
- Set `browserslist` to match your actual audience (avoids unnecessary polyfills)
- Audit dynamic imports that are never triggered
- Remove feature flags for permanently-enabled features

## Runtime performance

### The main thread

JavaScript runs on a single main thread shared with layout, painting, and user input. Any JavaScript that blocks this thread for > 50ms delays interaction response.

### Long tasks

A long task is any main thread task > 50ms. Long tasks directly degrade INP.

**Identify long tasks:**
- DevTools Performance panel → look for red corners on tasks
- `PerformanceObserver` with `"longtask"` entry type
- Lighthouse "Avoid long main-thread tasks" audit

**Break up long tasks:**

```javascript
// Using scheduler.yield() (Chromium 129+; feature-detect before use)
async function processItems(items) {
  for (const item of items) {
    processItem(item)
    if (globalThis.scheduler?.yield) {
      await scheduler.yield()
    }
  }
}

// Using setTimeout to yield to the event loop
function yieldToMain() {
  return new Promise(resolve => setTimeout(resolve, 0))
}

async function processItems(items) {
  const CHUNK_SIZE = 50
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE)
    for (const item of chunk) {
      processItem(item)
    }
    await yieldToMain()
  }
}

// Using requestIdleCallback for non-urgent work
function processInBackground(items) {
  let index = 0
  function processChunk(deadline) {
    while (index < items.length && deadline.timeRemaining() > 5) {
      processItem(items[index])
      index++
    }
    if (index < items.length) {
      requestIdleCallback(processChunk)
    }
  }
  requestIdleCallback(processChunk)
}
```

### Event handling

```javascript
// Debounce — wait until activity stops (search input, resize)
function debounce(fn, ms) {
  let timer
  return (...args) => {
    clearTimeout(timer)
    timer = setTimeout(() => fn(...args), ms)
  }
}

// Throttle — execute at most once per interval (scroll, mousemove)
function throttle(fn, ms) {
  let last = 0
  return (...args) => {
    const now = Date.now()
    if (now - last >= ms) {
      last = now
      fn(...args)
    }
  }
}
```

Guidelines:
- Debounce search/filter inputs (200-300ms)
- Throttle scroll handlers (16ms for 60fps, or use `IntersectionObserver` instead)
- Throttle resize handlers (100-200ms, or use `ResizeObserver`)
- Use `passive: true` on touch/wheel listeners that don't call `preventDefault()`
- Replace scroll-position polling with `IntersectionObserver`

### DOM manipulation

```javascript
// BAD — causes reflow on every iteration
for (const item of items) {
  container.appendChild(createCard(item))
}

// GOOD — batch with DocumentFragment
const fragment = document.createDocumentFragment()
for (const item of items) {
  fragment.appendChild(createCard(item))
}
container.appendChild(fragment)

// GOOD — batch with innerHTML (when content is trusted)
container.innerHTML = items.map(item => createCardHtml(item)).join("")
```

- Batch DOM writes using `DocumentFragment` or `innerHTML`
- Use `textContent` instead of `innerText` (innerText triggers layout)
- Detach elements from DOM before making many changes, then reattach
- Use `requestAnimationFrame` for visual updates

### List virtualization

Render only visible items + a small buffer instead of the full list.

When to virtualize:
- Lists with > 100 items
- Grids or tables with > 50 rows
- Any scrollable container where total DOM nodes > 500

Libraries: `@tanstack/virtual`, `react-window`, `react-virtuoso`

Key implementation details:
- Set explicit item heights when possible (avoids measurement)
- Overscan 5-10 items above/below viewport for smooth scrolling
- Recycle DOM nodes instead of creating/destroying them
- Handle dynamic item heights with a measurement cache

## Web Workers

Move CPU-intensive work off the main thread.

```javascript
// worker.ts
self.addEventListener("message", (event) => {
  const result = expensiveComputation(event.data)
  self.postMessage(result)
})

// main.ts
const worker = new Worker(new URL("./worker.ts", import.meta.url), { type: "module" })
worker.addEventListener("message", (event) => {
  updateUi(event.data)
})
worker.postMessage(inputData)
```

Good candidates for Web Workers:
- Data parsing (CSV, JSON, XML)
- Image processing (filters, resizing, encoding)
- Search indexing or filtering large datasets
- Cryptographic operations
- Complex calculations (statistics, ML inference)
- Markdown/rich text rendering

Not suitable for Workers:
- DOM access (Workers have no DOM)
- Quick operations (< 16ms) — overhead of message passing exceeds the gain
- Operations that need synchronous results

### Transferable objects

```javascript
// Avoid copying large data — transfer ownership instead
const buffer = new ArrayBuffer(1024 * 1024)
worker.postMessage(buffer, [buffer]) // buffer is now unusable in main thread
```

Use `Transferable` for `ArrayBuffer`, `MessagePort`, `ImageBitmap`, `OffscreenCanvas`.

## Memory management

### Common memory leaks

**Detached DOM nodes** — references to removed DOM elements prevent garbage collection:

```javascript
// BAD — reference keeps detached node alive
let cachedElement = document.getElementById("temp")
document.getElementById("temp").remove()
// cachedElement still references the detached node

// FIX — null the reference
cachedElement = null
```

**Event listeners not cleaned up:**

```javascript
// BAD — listener leaks if element is removed
element.addEventListener("click", handler)

// GOOD — use AbortController for cleanup
const controller = new AbortController()
element.addEventListener("click", handler, { signal: controller.signal })
// Later: controller.abort() removes all listeners registered with this signal
```

**Closures capturing large scopes:**

```javascript
// BAD — closure captures entire large array even though it only needs length
function createCounter(data) {
  const bigArray = processData(data) // 10MB array
  return () => bigArray.length // closure keeps bigArray alive forever
}

// FIX — capture only what you need
function createCounter(data) {
  const bigArray = processData(data)
  const len = bigArray.length
  return () => len
}
```

**Timers and intervals:**

```javascript
// BAD — interval runs forever if component unmounts
setInterval(pollServer, 5000)

// GOOD — clear on cleanup
const id = setInterval(pollServer, 5000)
// Later: clearInterval(id)
```

### Memory profiling

- DevTools Memory panel → Heap Snapshot for finding leaks
- DevTools Performance panel → Memory checkbox for allocation timeline
- `performance.measureUserAgentSpecificMemory()` for field measurement (Chromium-only, requires cross-origin isolation)
- Compare heap snapshots before and after an action to find leaks

## Common JavaScript performance issues to flag

- Full library imports where only one function is used
- Missing code splitting (entire app in a single bundle)
- Barrel files re-exporting heavy modules
- Long tasks (> 50ms) blocking interaction paths
- Scroll/resize handlers without throttle/debounce
- Layout thrashing (interleaved DOM reads and writes in loops)
- Large lists rendered without virtualization
- Event listeners not cleaned up on component unmount
- `setInterval` without corresponding `clearInterval`
- CPU-intensive work on the main thread that could use a Worker
- Missing `passive: true` on touch/wheel listeners
- Synchronous `localStorage`/`sessionStorage` access in hot paths
