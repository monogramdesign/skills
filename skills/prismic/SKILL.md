---
name: prismic
description: |
  Prismic CMS — Slice Machine, content modeling, rich text, previews, webhook revalidation, and TypeScript integration. Use when building or reviewing Prismic-powered sites.
  Triggers: "Prismic", "Slice Machine", "Prismic slices", "Prismic content model", "Prismic preview", "Prismic webhook"
lastReviewed: 2026-04-10
upstreamDeps: [prismic]
---

# Prismic

Use this skill when building or reviewing sites powered by Prismic — Slice Machine for component-driven content, content modeling, rich text rendering, preview mode, webhook-driven revalidation, and TypeScript integration.

## Core principles

1. **Slices are the building blocks** — Prismic's Slice Machine maps CMS components directly to frontend components. Every slice in the CMS should have a corresponding code component.
2. **Content team autonomy** — The architecture must let editors build pages from slices without engineering involvement. Design slices to be flexible enough for editors but constrained enough to maintain design consistency.
3. **Type-safe content** — Use Prismic's generated TypeScript types for all content queries. Never access content fields without type checking.
4. **Preview before publish** — Editors must be able to preview draft content in the actual site context before publishing.

## Slice Machine

### Setup
- Initialize with `npx @slicemachine/init`
- Slice Machine runs locally and syncs models to the Prismic repository
- Define slices in `slices/` directory with a `model.json` and component file
- Use `SliceZone` component to render a page's slice list

### Slice design
- Each slice has a `primary` zone (single fields) and an `items` zone (repeatable fields)
- Keep slices focused — one visual section per slice
- Use variations for different layouts of the same slice (e.g. "Default", "WithImage", "Centered")
- Name slices clearly: `HeroBanner`, `FeatureGrid`, `Testimonials`, `CallToAction`

### Component mapping
```typescript
import { SliceZone } from "@prismicio/react"
import { components } from "@/slices"

export default async function Page({ params }: { params: { uid: string } }) {
  const page = await client.getByUID("page", params.uid)
  return <SliceZone slices={page.data.slices} components={components} />
}
```

## Content modeling

### Custom types
- Use **Page types** for routable content (homepage, landing pages, blog posts)
- Use **Custom types** (non-repeatable) for singletons (site settings, navigation, footer)
- Use **Reusable types** for content referenced across pages (authors, categories)

### Field types
- **Rich Text** — for formatted content (headings, paragraphs, links, embeds)
- **Key Text** — for plain strings (titles, slugs, meta descriptions)
- **Image** — with responsive views for different breakpoints
- **Link** — internal (document link) or external (URL)
- **Content Relationship** — for referencing other documents
- **Group** — for repeatable field sets within a document
- **Slices** — for composable page sections

### Best practices
- Define image responsive views (thumbnail, mobile, desktop) in the content model
- Use content relationships instead of duplicating data across documents
- Keep custom types lean — use slices for page-level flexibility
- Name fields descriptively: `heroImage` not `image1`

## Rich text rendering

### Custom serializer
- Use `<PrismicRichText>` with a custom `components` prop for rendering
- Map Prismic rich text elements to your design system components
- Handle embedded entries (images, videos, code blocks) with custom renderers
- Handle internal links by resolving them to your routing structure

### Link resolution
- Define a `linkResolver` function that maps Prismic documents to URLs
- Use `<PrismicLink>` component for automatic link resolution
- Handle both internal document links and external URLs

## Preview mode

### Setup
- Create a preview API route that sets a preview cookie and redirects to the previewed page
- Create an exit-preview route that clears the cookie
- Configure the preview URL in the Prismic repository settings
- Use `enableAutoPreviews` in the Prismic client configuration

### Implementation
- Check for the preview cookie in data fetching functions
- When previewing, use the Preview API (draft content) instead of the CDN API (published)
- Show a visual indicator when preview mode is active
- Ensure preview works for both new (unpublished) and existing documents

## Webhook revalidation

### On-demand revalidation
- Set up a webhook endpoint that Prismic calls on content publish
- Verify the webhook secret before processing
- Call `revalidatePath` or `revalidateTag` for the affected pages
- Return a `200` response quickly — do heavy work asynchronously

### Cache strategy
- Use ISR with a long revalidation interval as a baseline
- Use on-demand revalidation via webhooks for immediate updates
- Tag cached content by document type for targeted revalidation

## TypeScript integration

- Run `npx prismic-ts-codegen` to generate types from your content models
- Regenerate types after every model change
- Use the generated types with `prismicio` client methods
- Type slice components with the generated slice types

## Review checklist

- Every slice has a corresponding frontend component
- `SliceZone` used for rendering page slices
- Content models use appropriate field types
- Rich text rendered with a custom serializer matching the design system
- Preview mode configured and working for editors
- Webhook revalidation set up for on-demand cache invalidation
- TypeScript types generated from content models
- Link resolver handles all document types
- Images use responsive views defined in the content model
- Content relationships used instead of data duplication

## Common issues to flag

- Slices without corresponding frontend components (render nothing)
- Rich text rendered with default serializer (doesn't match design system)
- Missing preview mode (editors can't preview before publishing)
- No webhook revalidation (content changes require a full rebuild)
- Untyped content queries (missing type generation)
- Hardcoded content that should come from Prismic
- Missing link resolver for internal links (broken navigation)
- Images without responsive views (serving desktop images on mobile)
- Content model changes not synced between Slice Machine and the repository
