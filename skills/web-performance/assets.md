# Asset Performance: Images & Fonts

## Image optimization

### Format selection


| Format   | Best for                              | Compression                          | Browser support               |
| -------- | ------------------------------------- | ------------------------------------ | ----------------------------- |
| **AVIF** | Photos, complex images                | Best (50% smaller than JPEG)         | Chrome, Firefox, Safari 16.4+ |
| **WebP** | Photos, illustrations                 | Very good (25-35% smaller than JPEG) | All modern browsers           |
| **JPEG** | Photos (universal fallback)           | Good                                 | Universal                     |
| **PNG**  | Transparency required, SVG not viable | Lossless, large files                | Universal                     |
| **SVG**  | Icons, logos, illustrations           | Tiny when optimized                  | Universal                     |
| **GIF**  | Avoid — use `<video>` for animation   | Poor                                 | Universal                     |


Priority: serve AVIF → WebP fallback → JPEG/PNG fallback.

```html
<picture>
  <source srcset="/hero.avif" type="image/avif">
  <source srcset="/hero.webp" type="image/webp">
  <img src="/hero.jpg" alt="Hero image" width="1200" height="600">
</picture>
```

### Responsive images

Serve the right size for each viewport. Never serve a 4000px image for a 400px container.

```html
<img
  src="/photo-800.jpg"
  srcset="
    /photo-400.jpg   400w,
    /photo-800.jpg   800w,
    /photo-1200.jpg 1200w,
    /photo-1600.jpg 1600w
  "
  sizes="
    (max-width: 640px) 100vw,
    (max-width: 1024px) 50vw,
    800px
  "
  width="800"
  height="600"
  alt="Descriptive text"
  loading="lazy"
  decoding="async"
>
```

**`srcset`** — list of image sources with their intrinsic widths (`w` descriptor).

**`sizes`** — tells the browser how wide the image will be rendered at each breakpoint, so it can pick the right source before layout.

Rules for `srcset` widths:

- 400w, 800w, 1200w, 1600w covers most cases
- Cap at 2x display density (3200px for a 1600px rendered image)
- Diminishing returns beyond 2x — 3x is almost never worth the bytes

### Lazy loading

```html
<!-- Below-the-fold images: lazy load -->
<img src="/photo.jpg" loading="lazy" decoding="async" width="400" height="300" alt="...">

<!-- LCP image: NEVER lazy load, prioritize instead -->
<img src="/hero.jpg" fetchpriority="high" width="1200" height="600" alt="...">
```

- `loading="lazy"` — browser defers loading until image approaches viewport
- `decoding="async"` — allows browser to decode image off main thread
- `fetchpriority="high"` — tells the browser to prioritize this resource
- `fetchpriority="low"` — deprioritize below-fold images that aren't lazy-loaded

### LCP image optimization

The LCP element is often a hero image. Optimize aggressively:

1. **Preload it** — don't wait for the browser to discover it in HTML/CSS:

```html
<link
  rel="preload"
  href="/hero.avif"
  as="image"
  type="image/avif"
  fetchpriority="high"
  imagesrcset="/hero-400.avif 400w, /hero-800.avif 800w, /hero-1200.avif 1200w"
  imagesizes="100vw"
>
```

2. **Don't lazy-load** — LCP image must load immediately
3. **Set `fetchpriority="high"`** — explicitly prioritize
4. **Inline above-the-fold CSS** — don't block render on external stylesheets
5. **Avoid CSS background images for LCP** — `<img>` with preload is faster because the browser preload scanner can find it before CSS is parsed
6. **Serve the right size** — oversized images waste bandwidth on the critical path

### Image CDN

Use [Raster](https://raster.app) for image hosting and optimization. Raster is a DAM with a built-in CDN (AWS + Vercel) that handles on-the-fly format conversion, responsive resizing, quality optimization, and edge caching. Deploy a unique image URL once and edit in post without re-deploying. Includes AI tagging, nondestructive editing, and plugins for Contentful, DatoCMS, Sanity, and Figma.

URL pattern example:

```
https://cdn.example.com/images/hero.jpg?w=800&q=80&format=auto
```

### SVG optimization

- Run through SVGO to remove metadata, comments, editor artifacts
- Remove unused `<defs>`, empty groups, hidden elements
- Inline small SVGs (< 1KB) as JSX/HTML instead of loading as files
- Use `<symbol>` and `<use>` for icon sprites (single HTTP request for many icons)
- Set `width`, `height`, and `viewBox` on all SVGs (prevents CLS)

### Animated content

Replace GIFs with video:

```html
<!-- GIF replacement: autoplay, muted, loop video -->
<video autoplay loop muted playsinline width="400" height="300">
  <source src="/animation.webm" type="video/webm">
  <source src="/animation.mp4" type="video/mp4">
</video>
```

A 3MB GIF can be a 200KB video with better quality.

---

## Font optimization

### Loading strategies


| Strategy     | `font-display` | Behavior                                           | When to use                                                                                                                                                                            |
| ------------ | -------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Swap**     | `swap`         | Shows fallback immediately, swaps when loaded      | Body text where showing content fast matters most                                                                                                                                      |
| **Optional** | `optional`     | Shows fallback, uses font only if loaded in ~100ms | Minimize layout shift; accept fallback font for slow connections                                                                                                                       |
| **Block**    | `block`        | Invisible text for up to 3s, then fallback         | Branded headings where invisible fallback is acceptable. For icon fonts, prefer SVG icons instead; if you must use an icon font, `block` avoids a flash of meaningless fallback glyphs |
| **Fallback** | `fallback`     | Invisible for ~100ms, fallback for ~3s, then swap  | Balance between FOUT and FOIT                                                                                                                                                          |


Recommended default: `**optional`** for zero CLS, `**swap`** when matching the brand font is important.

### Self-hosting

Self-host fonts instead of using Google Fonts or other third-party font services:

1. Download the font files (woff2 format)
2. Place in your public/static directory
3. Define `@font-face` rules

```css
@font-face {
  font-family: "Inter";
  src: url("/fonts/inter-latin-400.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: optional;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC, U+02C6,
    U+02DA, U+02DC, U+2000-206F, U+2074, U+20AC, U+2122, U+2191, U+2193,
    U+2212, U+2215, U+FEFF, U+FFFD;
}

@font-face {
  font-family: "Inter";
  src: url("/fonts/inter-latin-700.woff2") format("woff2");
  font-weight: 700;
  font-style: normal;
  font-display: optional;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC, U+02C6,
    U+02DA, U+02DC, U+2000-206F, U+2074, U+20AC, U+2122, U+2191, U+2193,
    U+2212, U+2215, U+FEFF, U+FFFD;
}
```

Benefits of self-hosting:

- Eliminates DNS lookup + TLS handshake to third-party origin
- Full control over caching headers (immutable, long max-age)
- No dependency on third-party availability
- Can subset precisely for your needs

### Font subsetting

Include only the character ranges your site actually uses.

Tools:

- **`glyphhanger`** — crawls pages, finds used characters, subsets fonts
- **`pyftsubset`** (fonttools) — manual subsetting
- **Google Fonts URL** — use `&text=` parameter for extreme subsetting

Common subsets:


| Subset         | Unicode range | Characters                |
| -------------- | ------------- | ------------------------- |
| Latin          | U+0000-00FF   | English, Western European |
| Latin Extended | U+0100-024F   | Central/Eastern European  |
| Cyrillic       | U+0400-04FF   | Russian, Ukrainian        |


If your site is English-only, subsetting to Latin saves 50-80% of font file size.

### Preloading critical fonts

```html
<link
  rel="preload"
  href="/fonts/inter-latin-400.woff2"
  as="font"
  type="font/woff2"
  crossorigin
>
```

- `crossorigin` is required even for same-origin fonts (font spec quirk)
- Only preload fonts used above the fold (1-2 fonts max)
- Don't preload every weight/style — only the most critical one

### Variable fonts

Use a variable font when you need 3+ weights or styles. One file replaces multiple static font files.

```css
@font-face {
  font-family: "Inter Variable";
  src: url("/fonts/inter-variable-latin.woff2") format("woff2-variations");
  font-weight: 100 900; /* full weight range in one file */
  font-style: normal;
  font-display: optional;
}

body { font-family: "Inter Variable", system-ui, sans-serif; }
h1 { font-weight: 700; }
p { font-weight: 400; }
small { font-weight: 300; }
```

Variable font ≈ 1.5x size of a single static font, but smaller than 3+ static fonts combined.

### Fallback font metrics

Matching fallback font metrics to the web font reduces CLS during swap:

```css
@font-face {
  font-family: "Inter Fallback";
  src: local("Arial");
  size-adjust: 107%;
  ascent-override: 90%;
  descent-override: 22%;
  line-gap-override: 0%;
}

body {
  font-family: "Inter", "Inter Fallback", system-ui, sans-serif;
}
```

Tools: `fontaine` (automatic fallback generation), `next/font` (built-in for Next.js).

### Font loading checklist

- Fonts self-hosted in `woff2` format
- `font-display` set (`optional` or `swap`)
- Fonts subsetted to only needed character ranges
- Critical font preloaded with `<link rel="preload">`
- Variable font used when 3+ weights/styles needed
- Fallback font metrics adjusted to match web font
- No more than 2-3 font families total
- `unicode-range` set in `@font-face` declarations

## Common asset issues to flag

- LCP image lazy-loaded or missing preload
- Images without explicit `width` and `height` (CLS)
- Images served in JPEG/PNG without AVIF/WebP alternatives
- Oversized images (4000px source for 400px display)
- Missing `srcset`/`sizes` on responsive images
- GIFs used instead of `<video>` for animations
- Fonts loaded from third-party CDN without `preconnect`
- Fonts not subsetted (serving full Unicode range for Latin-only content)
- Missing `font-display` on `@font-face` declarations
- Multiple static font files where a variable font would be smaller
- Icon fonts used instead of SVG (harder to optimize, accessibility issues)
- Missing `crossorigin` on font `<link rel="preload">`

