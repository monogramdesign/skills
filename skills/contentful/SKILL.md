---
name: contentful
description: |
  Contentful CMS — content types, rich text rendering, environments, migrations, webhook revalidation, and TypeScript integration. Use when building or reviewing Contentful-powered sites.
  Triggers: "Contentful", "content types", "rich text rendering", "Contentful migration", "Contentful webhook", "content model"
lastReviewed: 2026-04-10
upstreamDeps: [contentful]
---

# Contentful

Use this skill when building or reviewing sites powered by Contentful — content type modeling, rich text rendering, environments, migrations, webhook-driven revalidation, and TypeScript integration.

## Core principles

1. **Content model is architecture** — Content types define the structure of the entire site. Model them carefully around editorial workflows and frontend component needs, not database schemas.
2. **Environments for safety** — Use Contentful environments (sandbox, staging, master) to test content model changes before they affect production.
3. **Migrations as code** — Content model changes should be scripted, version-controlled, and applied through the Contentful Migration CLI. Never make structural changes manually in the web app for production.
4. **Two APIs, two purposes** — Content Delivery API (CDA) for published content, Content Preview API (CPA) for draft content. Use the right one for the right context.

## Content modeling

### Content types
- Model content types to match frontend components (a "Hero Banner" content type maps to a `HeroBanner` component)
- Use reference fields for relationships between content types (author → blog post)
- Use short text for titles and slugs, long text for body content
- Use JSON fields sparingly — prefer structured fields for queryable data

### Field types
- **Short text** — titles, slugs, labels (max 256 characters)
- **Long text** — body content (Markdown or plain text)
- **Rich text** — structured content with embedded entries and assets
- **Number** — integer or decimal
- **Date** — ISO 8601 date/time
- **Media** — images, videos, documents (stored as assets)
- **Reference** — links to other entries (single or multiple)
- **JSON** — arbitrary structured data (use sparingly)

### Best practices
- Use validation rules on fields (required, unique, regex patterns, character limits)
- Define appearance settings for a better editorial experience
- Use tags for cross-cutting categorization
- Keep content types focused — split large types into smaller, composable ones
- Use naming conventions: PascalCase for content types, camelCase for field IDs

## Rich text rendering

### Structure
- Contentful rich text is a structured JSON document (not HTML or Markdown)
- Use `@contentful/rich-text-react-renderer` (or equivalent for other frameworks)
- Define custom renderers for each node type

### Custom renderers
```typescript
import { documentToReactComponents } from "@contentful/rich-text-react-renderer"
import { BLOCKS, INLINES } from "@contentful/rich-text-types"

const options = {
  renderNode: {
    [BLOCKS.EMBEDDED_ENTRY]: (node) => {
      // Render embedded entries based on content type
    },
    [BLOCKS.EMBEDDED_ASSET]: (node) => {
      // Render images, videos, etc.
    },
    [INLINES.ENTRY_HYPERLINK]: (node) => {
      // Render internal links
    },
  },
}
```

### Embedded content
- Resolve embedded entries and assets in the same API call (use `include` parameter)
- Handle missing or unpublished embedded content gracefully
- Map embedded content types to the appropriate frontend components

## Environments and migrations

### Environments
- Use `master` for production content
- Create sandbox environments for testing content model changes
- Use environment aliases to swap environments without changing code
- Delete sandbox environments after changes are merged

### Migration CLI
```bash
contentful space migration --space-id <id> --environment-id <env> migration.js
```

- Write migrations as JavaScript files using the Contentful Migration API
- Version-control all migration files
- Test migrations in a sandbox environment before applying to master
- Migrations can create/modify/delete content types, fields, and entries

### Migration patterns
- Add new fields as optional first, then backfill data, then make required
- Never delete a field that's still referenced by the frontend
- Use `transformEntries` for data migrations
- Keep migrations idempotent where possible

## Data fetching

### Content Delivery API (CDA)
- Use for published content in production
- Cache responses aggressively (CDN, ISR, edge cache)
- Use `include` parameter to resolve linked entries in a single request (max depth 10)
- Use `select` parameter to fetch only needed fields (reduces payload size)
- Paginate with `skip` and `limit` (max 1000 entries per request)

### Content Preview API (CPA)
- Use for draft content in preview mode
- Same query interface as CDA but returns unpublished content
- Enable in preview/draft mode only — never in production

### Webhooks
- Configure webhooks for entry publish, unpublish, and delete events
- Use webhooks to trigger on-demand revalidation
- Verify webhook signatures (use the signing secret)
- Filter webhooks by content type to avoid unnecessary revalidation

## TypeScript integration

- Use `contentful-typescript-codegen` or `cf-content-types-generator` to generate types
- Regenerate types after every content model change
- Use generated types with the Contentful client for type-safe queries
- Create helper types for common query results

## Review checklist

- Content types match frontend component structure
- Rich text rendered with custom renderers matching the design system
- Environments used for testing content model changes
- Migrations scripted and version-controlled
- Webhook revalidation configured for content publish events
- TypeScript types generated from the content model
- Preview mode working for editors
- Linked entries resolved efficiently (using `include` parameter)
- Field validations configured for editorial guardrails
- Assets optimized with Contentful's image API (format, quality, dimensions)

## Common issues to flag

- Content model changes made manually in the web app (not through migrations)
- Rich text rendered with default renderer (doesn't match design system)
- Missing preview mode for editors
- No webhook revalidation (stale content after publish)
- Untyped content queries (missing type generation)
- Over-fetching with `include: 10` when fewer levels are needed
- Missing field validations (editors can enter invalid data)
- Hardcoded content that should come from Contentful
- Not using environment aliases (environment changes require code changes)
- Assets served without image API optimization (raw uploads)
