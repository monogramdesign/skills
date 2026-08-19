# Performance Monitoring

## Lab testing

Lab tests run in controlled conditions. Useful for debugging, comparing changes, and catching regressions before deployment.

### Lighthouse

Built into Chrome DevTools and available as CLI/CI tool.

```bash
# CLI audit
npx lighthouse https://example.com --output=json --output-path=./report.json

# CI-friendly: returns exit code based on thresholds
npx lighthouse https://example.com \
  --budget-path=budget.json \
  --output=json \
  --chrome-flags="--headless=new"
```

**Performance budget file (`budget.json`):**

```json
[
  {
    "resourceSizes": [
      { "resourceType": "script", "budget": 300 },
      { "resourceType": "stylesheet", "budget": 100 },
      { "resourceType": "image", "budget": 500 },
      { "resourceType": "font", "budget": 100 },
      { "resourceType": "total", "budget": 1000 }
    ],
    "resourceCounts": [
      { "resourceType": "third-party", "budget": 5 }
    ],
    "timings": [
      { "metric": "largest-contentful-paint", "budget": 2500 },
      { "metric": "cumulative-layout-shift", "budget": 0.1 },
      { "metric": "total-blocking-time", "budget": 300 },
      { "metric": "speed-index", "budget": 3000 }
    ]
  }
]
```

Limitations:
- Simulated throttling doesn't match real network conditions
- Single-page test doesn't capture navigation or interaction patterns
- Scores vary between runs (run 3-5 times and use median)

### WebPageTest

Deeper analysis than Lighthouse with real browsers, real devices, and real network conditions.

Key features:
- **Filmstrip view** — visual progression frame by frame
- **Waterfall chart** — detailed timing for every resource
- **Multi-step tests** — test user flows across multiple pages
- **Comparison view** — before/after side-by-side
- **Custom scripting** — automated interaction flows
- **Third-party breakdown** — impact of each third-party domain

Run from multiple locations to test geographic performance.

### Chrome DevTools Performance panel

For deep investigation of runtime performance:

1. **Record a trace** — click Record → interact with the page → Stop
2. **Flame chart** — identify long functions and hot paths
3. **Timings track** — FP, FCP, LCP, DCL markers
4. **Experience track** — layout shifts and long tasks
5. **Interactions track** — INP candidates with breakdown
6. **Layers panel** — GPU layer count and memory

Key shortcuts:
- Press `Ctrl+Shift+P` → "Show rendering" → check "Layout Shift Regions" and "Paint Flashing"
- "Coverage" tab shows unused JS/CSS bytes per file
- "Network request blocking" lets you simulate removing third-party scripts

## Field data (Real User Monitoring)

Field data captures actual user experience in production. Lab data alone is not enough — real users have diverse devices, networks, and usage patterns.

### Chrome User Experience Report (CrUX)

Free, public dataset of real-user CWV from Chrome users.

Access methods:
- **PageSpeed Insights** — instant CrUX lookup for a URL or origin
- **CrUX API** — programmatic access
- **CrUX Dashboard** (Looker Studio) — historical trends
- **BigQuery** — full dataset for advanced analysis

```bash
# CrUX API query
curl "https://chromeuxreport.googleapis.com/v1/records:queryRecord?key=API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/", "formFactor": "PHONE"}'
```

Limitations: 28-day rolling average, only pages with sufficient traffic, Chrome-only data.

### web-vitals library

Collect CWV from all users on your site (not just Chrome desktop).

```javascript
import { onLCP, onCLS, onINP, onFCP, onTTFB } from "web-vitals"

function sendMetric(metric) {
  const payload = {
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
    page: window.location.pathname,
    navigationType: metric.navigationType,
    // Include connection info for segmentation
    effectiveType: navigator.connection?.effectiveType,
    deviceMemory: navigator.deviceMemory,
  }
  navigator.sendBeacon("/api/vitals", JSON.stringify(payload))
}

onLCP(sendMetric)
onCLS(sendMetric)
onINP(sendMetric)
onFCP(sendMetric)
onTTFB(sendMetric)
```

### RUM analytics platforms

| Platform | Type | Notes |
|----------|------|-------|
| **Vercel Analytics** | Built-in | Automatic CWV tracking for Vercel-deployed apps |
| **Sentry Performance** | Paid | Full-stack tracing with CWV |
| **Datadog RUM** | Paid | Detailed session replay + performance |
| **SpeedCurve** | Paid | Synthetic + RUM, focused on performance |
| **Google Analytics 4** | Free | Basic CWV via web-vitals integration |
| **Custom solution** | DIY | web-vitals → your analytics pipeline |

### What to track in RUM

**Core metrics:**
- LCP, CLS, INP (Core Web Vitals)
- FCP, TTFB (supporting metrics)

**Segmentation dimensions:**
- Page/route
- Device type (mobile, desktop, tablet)
- Connection type (`navigator.connection.effectiveType`)
- Geography (from server logs)
- Browser/OS
- Navigation type (initial load, back/forward, reload, SPA navigation)
- User cohort (new vs returning, authenticated vs anonymous)

**Alerting thresholds:**
- Alert when p75 of any CWV crosses the "needs improvement" boundary
- Alert on sudden regression (> 20% degradation within 24h)
- Weekly report on CWV trends by page

## Performance budgets

### Types of budgets

| Budget type | What it measures | Example |
|-------------|-----------------|---------|
| **Size budget** | Transfer size of resources | Total JS < 300KB, Total CSS < 100KB |
| **Count budget** | Number of requests | < 50 requests per page, < 5 third-party origins |
| **Timing budget** | Metric thresholds | LCP < 2.5s, INP < 200ms, TBT < 300ms |

### Setting budgets

1. **Baseline**: measure current performance on key pages
2. **Competitive analysis**: benchmark against 2-3 competitors
3. **Target**: set budgets 20% better than current baseline, or match the best competitor
4. **Enforce**: fail CI builds that exceed budgets

### Budget enforcement in CI

**Using Lighthouse CI:**

`.lighthouserc.json`:

```json
{
  "ci": {
    "collect": {
      "url": ["http://localhost:3000/", "http://localhost:3000/products"],
      "numberOfRuns": 3
    },
    "assert": {
      "assertions": {
        "categories:performance": ["error", { "minScore": 0.9 }],
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }],
        "total-blocking-time": ["error", { "maxNumericValue": 300 }],
        "speed-index": ["error", { "maxNumericValue": 3000 }],
        "resource-summary:script:size": ["error", { "maxNumericValue": 300000 }]
      }
    }
  }
}
```

**Using bundlesize:**

```json
{
  "bundlesize": [
    { "path": "dist/js/*.js", "maxSize": "150 kB" },
    { "path": "dist/css/*.css", "maxSize": "30 kB" }
  ]
}
```

**Using size-limit:**

```json
[
  { "path": "dist/index.js", "limit": "50 kB" },
  { "path": "dist/vendor.js", "limit": "150 kB" }
]
```

### CI pipeline integration

```yaml
# GitHub Actions example
performance-check:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - run: npm ci && npm run build
    - name: Bundle size check
      run: npx size-limit
    - name: Lighthouse CI
      run: |
        npm start &
        npx lhci autorun
```

### Performance regression workflow

1. **PR check**: run Lighthouse + bundle size on every PR
2. **Compare to main**: show delta in PR comment (size-limit, Lighthouse CI, or bundlewatch do this)
3. **Block merge**: if any budget is exceeded, mark the check as failed
4. **Dashboard**: track trends over time on a shared dashboard
5. **Alerting**: notify on-call when field metrics regress

## Continuous monitoring checklist

- [ ] web-vitals library sending LCP, CLS, INP to analytics
- [ ] Data segmented by page, device type, and connection
- [ ] Alerts configured for CWV threshold crossings
- [ ] Lighthouse CI running on every PR
- [ ] Bundle size budget enforced in CI
- [ ] Performance budget set for total JS, CSS, and image size
- [ ] CrUX data reviewed monthly
- [ ] Third-party script impact audited quarterly
- [ ] Performance trends visible on a shared dashboard
- [ ] Regression investigation runbook documented

## Common monitoring issues to flag

- No field data collection (only relying on lab tests)
- CWV collected but not segmented by device/connection/page
- No alerting on performance regression
- Lighthouse CI not integrated in PR workflow
- No bundle size budget (size creep goes unnoticed)
- Performance data collected but never reviewed
- Missing TTFB monitoring (server issues go undetected)
- Only tracking averages, not percentiles (p75, p95)
