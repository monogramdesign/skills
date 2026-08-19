---
name: datocms
description: |
  DatoCMS — structured text, modular content, GraphQL API, real-time previews, image optimization, and TypeScript integration. Use when building or reviewing DatoCMS-powered sites.
  Triggers: "DatoCMS", "structured text", "modular content", "DatoCMS GraphQL", "DatoCMS preview", "Dato"
lastReviewed: 2026-04-10
upstreamDeps: [datocms]
---

# DatoCMS

Use this skill when building or reviewing sites powered by DatoCMS — structured text rendering, modular content blocks, GraphQL API, real-time previews, image optimization, and TypeScript integration.

## Core principles

1. **GraphQL-native** — DatoCMS exposes a GraphQL API as the primary data access layer. Use it for all content queries with full type safety.
2. **Structured Text over rich text** — DatoCMS uses Structured Text (a portable, structured format) instead of HTML or Markdown. Render it with custom components for full control.
3. **Image optimization built-in** — DatoCMS provides an image CDN with automatic responsive images, format conversion, and blur-up placeholders. Use it instead of processing images yourself.
4. **Modular content for flexibility** — Use modular content fields (blocks) to give editors composable page-building capabilities while maintaining design consistency.

## Content modeling

### Models
- Use models for each content type (pages, blog posts, products, authors)
- Use block models for reusable content blocks within modular content fields
- Use single-instance models for singletons (site settings, navigation)

### Field types
- **Single-line string** — titles, labels, slugs
- **Multi-line text** — plain text descriptions
- **Structured Text** — rich content with embedded records and blocks
- **Modular Content** — composable blocks for page building
- **Image / File** — media with automatic CDN optimization
- **Link** — references to other records (single or multiple)
- **JSON** — arbitrary structured data
- **SEO** — built-in SEO metadata field (title, description, image)
- **Slug** — URL-friendly identifier with auto-generation
- **Color** — color picker with hex/rgb output

### Best practices
- Use the built-in SEO field type for meta tags (title, description, social image)
- Define validations on fields (required, unique, format, character limits)
- Use fieldsets to organize complex models in the editor
- Keep models focused — use modular content for page-level flexibility
- Use meaningful API identifiers (they become GraphQL field names)

## GraphQL API

### Querying
```graphql
query BlogPosts {
  allPosts(orderBy: publishedAt_DESC, first: 10) {
    id
    title
    slug
    publishedAt
    author {
      name
      avatar {
        responsiveImage(imgixParams: { w: 48, h: 48, fit: crop }) {
          ...responsiveImageFragment
        }
      }
    }
    content {
      value
      blocks {
        __typename
        ... on ImageBlockRecord {
          image { responsiveImage { ...responsiveImageFragment } }
        }
      }
    }
  }
}
```

### Patterns
- Use `first` and `skip` for pagination
- Use `orderBy` for sorting (field name + `_ASC` or `_DESC`)
- Use `filter` for querying by field values
- Use `responsiveImage` fragment for optimized images
- Fetch only needed fields (GraphQL naturally prevents over-fetching)

### Draft content
- Use the `includeDrafts: true` header for preview mode
- Use the `excludeInvalid: true` header to filter out invalid draft records
- Switch between published and draft APIs based on preview mode

## Structured Text

### Rendering
- Use `react-datocms` (or equivalent) to render Structured Text
- Define custom renderers for each block type and inline record

```typescript
import { StructuredText } from "react-datocms"

<StructuredText
  data={post.content}
  renderBlock={({ record }) => {
    switch (record.__typename) {
      case "ImageBlockRecord":
        return <ImageBlock {...record} />
      case "CodeBlockRecord":
        return <CodeBlock {...record} />
      default:
        return null
    }
  }}
  renderInlineRecord={({ record }) => {
    // Render inline linked records
  }}
  renderLinkToRecord={({ record, children }) => {
    // Render links to other records
  }}
/>
```

### Best practices
- Handle all block types — return `null` for unknown types rather than crashing
- Map blocks to design system components
- Handle inline records and record links for internal navigation

## Image optimization

### Responsive images
- Use `responsiveImage` in GraphQL queries for automatic srcset, sizes, and blur-up
- Use the `<Image>` component from `react-datocms` for lazy loading and blur-up placeholders
- Configure `imgixParams` for transformations (resize, crop, format, quality)

### Best practices
- Always use `responsiveImage` instead of raw `url` for images
- Set appropriate `imgixParams` for each use case (thumbnails, hero images, avatars)
- Use `auto: format` for automatic WebP/AVIF delivery
- Leverage the built-in LQIP (Low Quality Image Placeholder) for blur-up loading

## Real-time previews

### Setup
- Use DatoCMS Real-time Updates API for live preview in the frontend
- Configure preview mode in your framework (draft mode)
- Use `useQuerySubscription` hook from `react-datocms` for real-time updates

### Implementation
- Create a preview API route that enables draft mode and redirects
- In preview mode, use `includeDrafts: true` header
- Show a visual indicator when preview mode is active
- Provide an exit-preview route

## Webhook revalidation

- Configure webhooks in DatoCMS project settings
- Filter by record type and event (publish, unpublish, delete)
- Verify webhook signatures with the shared secret
- Trigger on-demand revalidation for affected pages
- Use `X-DatoCMS-Webhook-Token` header for verification

## TypeScript integration

- Use `datocms-structured-text-utils` for Structured Text type utilities
- Generate TypeScript types from the GraphQL schema with GraphQL Code Generator
- Use typed GraphQL queries for full type safety
- Regenerate types after every model change

## Review checklist

- Content models use appropriate field types with validations
- GraphQL queries fetch only needed fields
- Structured Text rendered with custom components matching the design system
- Images use `responsiveImage` with appropriate `imgixParams`
- Preview mode configured for editors
- Webhook revalidation set up for content publish events
- TypeScript types generated from the GraphQL schema
- SEO field type used for meta tags
- Modular content blocks mapped to frontend components
- Real-time preview updates working for editors

## Common issues to flag

- Using raw image `url` instead of `responsiveImage` (no optimization)
- Structured Text rendered with default components (doesn't match design system)
- Missing preview mode for editors
- No webhook revalidation (stale content after publish)
- Untyped GraphQL queries (missing code generation)
- Missing field validations in content models
- Hardcoded content that should come from DatoCMS
- Not using the built-in SEO field type (reinventing meta tag management)
- GraphQL queries fetching unnecessary fields or relations
- Missing block type handlers in Structured Text rendering (silent failures)
