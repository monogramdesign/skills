# Network & Caching Performance

## Resource hints

Resource hints tell the browser about resources it will need soon, before it discovers them naturally.

```html
<!-- dns-prefetch: resolve DNS for a third-party origin (cheapest hint) -->
<link rel="dns-prefetch" href="https://cdn.example.com">

<!-- preconnect: DNS + TCP + TLS for a critical third-party origin -->
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<!-- preload: fetch a specific resource needed for current page -->
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/hero.avif" as="image" type="image/avif">
<link rel="preload" href="/critical.css" as="style">

<!-- prefetch: fetch a resource likely needed for the next navigation -->
<link rel="prefetch" href="/next-page.js">

<!-- modulepreload: preload + parse + compile an ES module -->
<link rel="modulepreload" href="/app.js">
```

### When to use each hint

| Hint | Cost | Use when |
|------|------|----------|
| `dns-prefetch` | Very low | You know the user will hit a third-party origin soon |
| `preconnect` | Low-medium | A critical third-party origin needed for the current page (limit to 2-4 origins) |
| `preload` | Medium | A resource needed for first render but not discoverable from HTML (fonts from CSS, images from JS) |
| `prefetch` | Low (idle) | A resource very likely needed on the next navigation |
| `modulepreload` | Medium | An ES module needed for current page (preloads + parses + compiles) |

### Common mistakes

- **Too many `preconnect`** — each connection costs memory and CPU; limit to 2-4 origins
- **Preloading unused resources** — every preload that isn't used within 3s triggers a console warning and wastes bandwidth
- **Missing `as` attribute** on preload — browser can't set priority correctly, may download twice
- **Missing `crossorigin` on font preload** — fonts always require CORS; without `crossorigin`, the preloaded file is discarded and re-fetched
- **Preloading resources behind dynamic conditions** — don't preload resources that may not be used

## HTTP caching

### Cache-Control header

```
Cache-Control: [visibility] [max-age] [revalidation] [immutable]
```

| Directive | Meaning |
|-----------|---------|
| `public` | Any cache (CDN, browser) can store this |
| `private` | Only the browser can cache this (not CDN) |
| `no-store` | Never cache this response |
| `no-cache` | Cache but revalidate with server every time |
| `max-age=N` | Fresh for N seconds from download |
| `s-maxage=N` | CDN-specific max-age (overrides `max-age` for shared caches) |
| `stale-while-revalidate=N` | Serve stale while fetching fresh copy in background for N seconds |
| `immutable` | Resource will never change (use with hashed filenames) |
| `must-revalidate` | When stale, must check with server before using |

### Caching strategies by resource type

| Resource | Cache-Control | Notes |
|----------|---------------|-------|
| Hashed JS/CSS (`app.a1b2c3.js`) | `public, max-age=31536000, immutable` | Content-hash in filename = safe to cache forever |
| Unhashed JS/CSS | `public, max-age=0, must-revalidate` | Or use `no-cache` |
| HTML pages | `public, max-age=0, must-revalidate` | Always revalidate HTML (entry point for fresh assets) |
| Static images (with hash) | `public, max-age=31536000, immutable` | Same as hashed JS/CSS |
| Static images (no hash) | `public, max-age=86400` | 24h cache, adjust based on change frequency |
| API (public, cacheable) | `public, s-maxage=60, stale-while-revalidate=300` | CDN caches for 60s, serves stale up to 5min while revalidating |
| API (private/personalized) | `private, no-store` | User-specific data must not be cached by CDN |
| Fonts | `public, max-age=31536000, immutable` | Fonts don't change; use versioned URLs |

### ETag and conditional requests

When `max-age` expires, the browser sends a conditional request:

```
GET /style.css
If-None-Match: "abc123"     ← ETag from previous response
If-Modified-Since: Thu, 01 Jan 2026 00:00:00 GMT
```

Server responds with `304 Not Modified` (no body) if unchanged, saving bandwidth.

- **ETag** — content-based hash; most accurate
- **Last-Modified** — timestamp; less precise but widely supported
- Both avoid re-downloading unchanged resources but still require a network roundtrip

### Vary header

```
Vary: Accept-Encoding, Accept-Language
```

`Vary` tells caches that the response differs based on these request headers. CDNs store separate cached copies per variant.

- `Vary: Accept-Encoding` — standard; different response for gzip vs brotli
- `Vary: Accept-Language` — for locale-specific responses
- Never `Vary: *` or `Vary: User-Agent` — effectively disables caching (too many variants)
- `Vary: Cookie` — use carefully; creates per-user cache entries

## Compression

### Algorithms

| Algorithm | Compression ratio | Speed | Browser support |
|-----------|-------------------|-------|-----------------|
| **Brotli** (`br`) | Best (15-25% smaller than gzip) | Slower to compress, same decompression | All modern browsers |
| **Gzip** (`gzip`) | Good | Fast | Universal |
| **Zstandard** (`zstd`) | Better than gzip, similar to Brotli | Fastest | Chrome 123+, Firefox, limited |

Serve Brotli with gzip fallback. Most CDNs and web servers handle this automatically via `Accept-Encoding` negotiation.

### What to compress

Compress text-based resources:
- HTML, CSS, JavaScript
- JSON, XML, SVG
- Web fonts (woff2 has built-in compression — don't double-compress)

Don't compress already-compressed formats:
- JPEG, PNG, WebP, AVIF
- Video (mp4, webm)
- woff2 (already Brotli-compressed internally)

### Static vs dynamic compression

- **Static** (build-time): pre-compress assets to Brotli level 11 (maximum). Use for JS, CSS, SVG.
- **Dynamic** (request-time): compress on the fly at lower level (Brotli 4-6). Use for HTML, API responses.

Static compression is much better quality (higher compression ratio) because there's no latency constraint.

## HTTP/2 and HTTP/3

### HTTP/2

- **Multiplexing** — multiple requests over a single connection (no head-of-line blocking at HTTP level)
- **Header compression** (HPACK) — reduces repeated header overhead
- **Server Push** (deprecated in most implementations) — avoid; use `<link rel="preload">` instead

Implications for optimization:
- **Prefer splitting over mega-bundles** — multiplexing handles many smaller files well, but bundling still helps compression ratio and cache granularity; find the right balance for your app
- **Sprites are rarely worth it** — individual images with proper caching are usually preferable, though icon sprite sheets can still win for many small icons
- **Don't shard domains** — one connection per origin is optimal with HTTP/2

### HTTP/3 (QUIC)

- **No head-of-line blocking at transport level** — packet loss on one stream doesn't block others
- **Faster connection setup** — 0-RTT or 1-RTT handshake (vs 2-3 RTT for TCP+TLS)
- **Connection migration** — survives network changes (mobile switching WiFi ↔ cellular)

Enable HTTP/3 at the CDN/server level. No code changes needed.

## CDN configuration

### Caching rules

- **Cache static assets at the edge** with content-hash filenames and long TTLs
- **Cache HTML with short TTL** (`s-maxage=60`) or `stale-while-revalidate`
- **Purge on deploy** — clear CDN cache for HTML and unhashed assets after each deployment
- **Tiered caching** — CDN → origin shield → origin server reduces origin load

### Edge optimization

- **Edge compression** — CDN compresses responses if origin doesn't
- **Image optimization at the edge** — format conversion, resizing via URL parameters
- **Early hints** (103) — CDN sends `Link: preload` headers before origin responds

```
HTTP/1.1 103 Early Hints
Link: </style.css>; rel=preload; as=style
Link: </fonts/inter.woff2>; rel=preload; as=font; crossorigin
```

### Geographic distribution

- Measure TTFB from target geographies (use WebPageTest with different locations)
- Ensure CDN has PoPs near your primary user base
- Consider edge compute (Cloudflare Workers, Vercel Edge Functions) for personalized responses

## Service workers

### Caching strategies

| Strategy | How it works | Use for |
|----------|-------------|---------|
| **Cache first** | Check cache → return if found → fetch if miss | Static assets, fonts, images |
| **Network first** | Try network → fall back to cache | API calls, fresh content |
| **Stale while revalidate** | Return cache immediately → update cache from network | Content that can be slightly stale |
| **Network only** | Always fetch from network | Real-time data, authentication |
| **Cache only** | Only serve from cache | Precached app shell |

### Implementation

```javascript
// Cache-first strategy
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      if (cached) return cached
      return fetch(event.request).then((response) => {
        const clone = response.clone()
        caches.open("v1").then((cache) => cache.put(event.request, clone))
        return response
      })
    })
  )
})
```

### Precaching

Precache the app shell and critical assets on service worker install:

```javascript
const PRECACHE = ["/", "/app.js", "/style.css", "/offline.html"]

self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open("precache-v1").then((cache) => cache.addAll(PRECACHE))
  )
})
```

### Service worker pitfalls

- **Don't cache HTML aggressively** — users may see stale content indefinitely
- **Version your caches** — delete old caches on activation
- **Keep the service worker small** — it runs on every page load (parse + execute cost)
- **Don't precache too much** — only cache what's needed for offline or instant repeat visits
- **Handle updates** — `skipWaiting()` + `clients.claim()` for immediate activation, or prompt user

## Third-party scripts

### Impact assessment

For each third-party script, evaluate:
1. **Size** — total transfer size (JS + any resources it loads)
2. **Blocking time** — does it block rendering or main thread?
3. **Network requests** — how many additional requests does it trigger?
4. **Privacy/security** — what data does it collect? Is it GDPR-compliant?
5. **Necessity** — is this actually needed? Can it be removed?

### Loading strategies

```html
<!-- Best: async, doesn't block parsing or rendering -->
<script src="https://analytics.example.com/script.js" async></script>

<!-- Good: defer, executes after parsing but before DOMContentLoaded -->
<script src="https://widget.example.com/sdk.js" defer></script>

<!-- Best for heavy embeds: load on interaction (facade pattern) -->
<button onclick="loadChatWidget()">Open Chat</button>
```

### Facade pattern

Replace heavy embeds with a lightweight placeholder that loads the real embed on interaction:

```html
<!-- Instead of loading a 1MB YouTube embed on page load -->
<div class="youtube-facade" onclick="loadYouTube(this)" data-video-id="abc123">
  <img src="/youtube-thumbnail-abc123.jpg" alt="Video title" width="640" height="360">
  <button aria-label="Play video">▶</button>
</div>
```

Apply facades to:
- YouTube/Vimeo embeds
- Google Maps
- Chat widgets (Intercom, Drift, Crisp)
- Social media embeds (Twitter, Instagram)
- Comment systems (Disqus)

### Self-hosting third-party scripts

Self-host scripts when possible to eliminate third-party connection overhead:

1. Download the script
2. Serve from your own domain
3. Set up a scheduled job to check for updates
4. Cache with your own headers

Benefits: no DNS/TLS overhead, full cache control, resilient to third-party outages.
Drawbacks: you own updates, some scripts detect and block self-hosting.

### Performance budget for third-party scripts

Set and enforce limits:
- Total third-party JS: < 100KB compressed
- No single third-party script > 50KB compressed
- Total third-party blocking time: < 200ms
- Maximum third-party origins: 3-5

## Common network issues to flag

- Missing `preconnect` for critical third-party origins
- Preloading resources that are never used
- Missing `crossorigin` on font preloads
- Hashed assets without `immutable` cache header
- HTML cached with long TTL (users see stale content)
- No compression configured (missing Brotli/gzip)
- Third-party scripts loaded synchronously
- Heavy embeds without facade pattern
- Too many third-party origins (each needs DNS + TLS)
- Missing CDN for static assets
- No `stale-while-revalidate` on frequently accessed content
- Service worker precaching too many resources
- `Vary: User-Agent` or `Vary: Cookie` destroying cache hit rate
