---
name: sanity
description: |
  Sanity CMS — GROQ queries, Studio customization, structured content, real-time previews, Visual Editing, and TypeScript integration. Use when building or reviewing Sanity-powered sites.
  Triggers: "Sanity", "GROQ", "Sanity Studio", "structured content", "Visual Editing", "Sanity schema", "Sanity preview"
lastReviewed: 2026-04-10
upstreamDeps: [sanity]
---

# Sanity

Use this skill when building or reviewing sites powered by Sanity — GROQ queries, Studio customization, structured content modeling, real-time previews, Visual Editing, and TypeScript integration.

## Core principles

1. **Structured content, not pages** — Sanity treats content as structured data, not page layouts. Model content for reuse across channels (web, mobile, email), not for a single page template.
2. **GROQ is the query language** — GROQ is Sanity's native query language. It's powerful, composable, and purpose-built for content queries. Prefer it over the GraphQL API.
3. **Real-time by default** — Sanity's Content Lake supports real-time updates. Use this for live previews and collaborative editing.
4. **Studio is customizable** — Sanity Studio is a React application. Customize it with plugins, custom inputs, and document actions to match the editorial workflow.

## Content modeling

### Schema definition
- Define schemas in code (TypeScript/JavaScript files)
- Use `document` type for top-level content (pages, posts, authors)
- Use `object` type for reusable field groups (address, SEO metadata)
- Use `array` of blocks for rich text (Portable Text)

### Field types
- **string** — short text (titles, labels)
- **text** — multi-line plain text
- **block** — Portable Text (rich text with custom blocks)
- **image** — with hotspot and crop metadata
- **reference** — links to other documents
- **slug** — URL-friendly identifier with auto-generation
- **array** — lists of any type (strings, objects, references)
- **object** — nested field groups

### Best practices
- Use `validation` rules on fields (required, min/max length, custom validators)
- Define `initialValue` for sensible defaults
- Use `fieldsets` and `groups` to organize complex document types in the Studio
- Use `orderings` to define default sort orders for document lists
- Keep schemas modular — extract reusable objects into separate files

## GROQ queries

### Basics
```groq
// Fetch all published blog posts with author details
*[_type == "post" && !(_id in path("drafts.**"))]{
  title,
  slug,
  publishedAt,
  "author": author->{name, image},
  body
} | order(publishedAt desc)
```

### Patterns
- Use projections to fetch only needed fields
- Use `->` for dereferencing references (joins)
- Use `| order()` for sorting
- Use `[0...10]` for pagination (offset-based)
- Use `count()` for totals
- Use `defined()` to filter for non-null fields

### Performance
- Fetch only the fields you need (avoid `*[_type == "post"]` without projections)
- Use `_id` and `_rev` for cache invalidation
- Cache GROQ results with ISR or edge caching
- Use `groq` tagged template literal for syntax highlighting and tooling

## Sanity Studio

### Customization
- Add custom input components for specialized fields (color pickers, map selectors)
- Use document actions for custom workflows (approve, schedule, archive)
- Use document badges to show status indicators
- Configure the Structure Builder for custom navigation in the Studio

### Plugins
- Use `@sanity/vision` for testing GROQ queries in the Studio
- Use `@sanity/dashboard` for editorial dashboards
- Use community plugins for common needs (SEO, media library, i18n)

## Portable Text (rich text)

### Rendering
- Use `@portabletext/react` (or equivalent for other frameworks) to render Portable Text
- Define custom components for each block type and mark
- Handle custom blocks (code snippets, embeds, callouts) with dedicated components

```typescript
import { PortableText } from "@portabletext/react"

const components = {
  types: {
    image: ({ value }) => <SanityImage {...value} />,
    code: ({ value }) => <CodeBlock {...value} />,
  },
  marks: {
    link: ({ children, value }) => <a href={value.href}>{children}</a>,
    internalLink: ({ children, value }) => <Link href={resolveRef(value)}>{children}</Link>,
  },
}
```

## Previews and Visual Editing

### Draft previews
- Use Sanity's Presentation tool for live previews in the Studio
- Configure `draftMode` in your framework for rendering draft content
- Use the `@sanity/preview-url-secret` package for secure preview URLs
- Show a visual indicator when viewing draft content

### Visual Editing
- Enable Visual Editing with `@sanity/visual-editing` for click-to-edit overlays
- Annotate content with `studioUrl` and `studioPath` for direct links to the Studio
- Use `encodeDataAttribute` to add edit metadata to rendered content
- Works with both draft and published content

### Real-time updates
- Use Sanity's listener API for real-time content updates during preview
- Subscribe to document changes with `client.listen()`
- Update the UI immediately when editors make changes

## Webhook revalidation

- Configure webhooks in the Sanity project settings
- Filter webhooks by document type to avoid unnecessary revalidation
- Verify webhook signatures with the shared secret
- Trigger on-demand revalidation (`revalidatePath` or `revalidateTag`)
- Return `200` quickly — process heavy work asynchronously

## TypeScript integration

- Use `sanity typegen` to generate TypeScript types from your schema
- Use `groq` tagged template literals for type-safe GROQ queries
- Regenerate types after every schema change
- Use generated types in both the Studio and the frontend

## Image handling

- Use `@sanity/image-url` for building optimized image URLs
- Leverage Sanity's image CDN for on-the-fly transformations (resize, crop, format)
- Use hotspot and crop data from the image field for responsive images
- Implement blur-up placeholders with low-quality image previews (LQIP)

## Review checklist

- Content schemas use appropriate field types and validations
- GROQ queries fetch only needed fields (no over-fetching)
- Portable Text rendered with custom components matching the design system
- Preview mode configured for editors (draft content visible)
- Visual Editing enabled for click-to-edit workflow
- Webhook revalidation configured for content publish events
- TypeScript types generated from schemas
- Images use Sanity's CDN for optimization
- Studio customized for the editorial workflow
- References resolved efficiently in GROQ queries

## Common issues to flag

- GROQ queries without projections (fetching all fields)
- Portable Text rendered with default components (doesn't match design system)
- Missing preview mode for editors
- No webhook revalidation (stale content after publish)
- Untyped GROQ queries (missing type generation)
- Images served without CDN optimization (raw asset URLs)
- Missing field validations in schemas
- Hardcoded content that should come from Sanity
- References not dereferenced in queries (returning `_ref` instead of the actual data)
- Studio not customized for the editorial workflow (confusing for editors)
