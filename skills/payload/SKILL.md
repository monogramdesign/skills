---
name: payload
description: |
  Payload CMS — collections, globals, access control, hooks, rich text, self-hosted deployment, and TypeScript integration. Use when building or reviewing Payload-powered sites.
  Triggers: "Payload", "Payload CMS", "Payload collections", "Payload access control", "Payload hooks", "self-hosted CMS"
lastReviewed: 2026-04-10
upstreamDeps: [payload]
---

# Payload CMS

Use this skill when building or reviewing sites powered by Payload — collections, globals, access control, hooks, rich text (Lexical), self-hosted deployment, and TypeScript-first configuration.

## Core principles

1. **Config is code** — Payload's entire configuration is TypeScript. Collections, fields, access control, hooks, and admin UI are all defined in code, version-controlled, and type-safe.
2. **Self-hosted by default** — Payload runs in your own infrastructure (Node.js server). You own the data, the deployment, and the scaling. No vendor lock-in.
3. **Access control is granular** — Define access control at the collection, field, and operation level. Every read, create, update, and delete can have its own access function.
4. **Local API for server-side** — Use Payload's Local API for server-side data fetching (zero HTTP overhead). Use the REST or GraphQL APIs for client-side or external access.

## Collections

### Definition
```typescript
import type { CollectionConfig } from "payload"

export const Posts: CollectionConfig = {
  slug: "posts",
  admin: {
    useAsTitle: "title",
    defaultColumns: ["title", "status", "publishedAt"],
  },
  fields: [
    { name: "title", type: "text", required: true },
    { name: "slug", type: "text", required: true, unique: true },
    { name: "content", type: "richText" },
    { name: "author", type: "relationship", relationTo: "users" },
    { name: "status", type: "select", options: ["draft", "published"] },
    { name: "publishedAt", type: "date" },
  ],
}
```

### Best practices
- Use `slug` as the collection identifier (kebab-case, plural)
- Set `useAsTitle` for a meaningful display in the admin panel
- Define `defaultColumns` for the list view
- Use `versions` and `drafts` for content that needs a publish workflow
- Group related fields with `tabs`, `collapsible`, or `row` layouts

## Globals

- Use globals for singleton content (site settings, navigation, footer)
- Define access control on globals just like collections
- Globals have a single document — no slug or list view

## Field types

- **text** — short strings
- **textarea** — multi-line plain text
- **richText** — Lexical editor (block-based rich text)
- **number** — integer or float
- **date** — date/time picker
- **upload** — file uploads with image optimization
- **relationship** — references to other collections
- **array** — repeatable field groups
- **blocks** — composable content blocks (similar to slices)
- **group** — nested field groups
- **select** / **radio** — enumerated options
- **checkbox** — boolean
- **json** — arbitrary JSON (use sparingly)

## Access control

### Collection-level
```typescript
access: {
  read: ({ req: { user } }) => {
    if (user?.role === "admin") return true
    return { status: { equals: "published" } }
  },
  create: ({ req: { user } }) => user?.role === "admin",
  update: ({ req: { user } }) => user?.role === "admin",
  delete: ({ req: { user } }) => user?.role === "admin",
}
```

### Field-level
- Restrict sensitive fields with field-level access control
- Use `hidden` to hide fields from the admin UI
- Use `admin.readOnly` for display-only fields

### Patterns
- Return `true` to allow, `false` to deny
- Return a query constraint to filter results (e.g. only published posts)
- Access functions receive the request, user, and document data
- Keep access functions fast — they run on every request

## Hooks

### Collection hooks
- `beforeChange` — validate or transform data before save
- `afterChange` — trigger side effects (revalidation, notifications, sync)
- `beforeDelete` — clean up related data
- `afterRead` — transform data before it's returned

### Field hooks
- `beforeValidate` — transform field value before validation
- `beforeChange` — transform before save
- `afterRead` — transform on read (e.g. compute derived fields)

### Best practices
- Keep hooks focused — one side effect per hook
- Use `afterChange` for cache revalidation
- Handle errors gracefully — a failing hook shouldn't break the save
- Use the `operation` parameter to differentiate between create and update

## Rich text (Lexical)

- Payload uses Lexical as the default rich text editor
- Configure custom blocks, inline elements, and marks
- Use the `lexicalHTML` field for server-rendered HTML output
- Define custom Lexical features for specialized content (code blocks, callouts, embeds)

## Local API

```typescript
const posts = await payload.find({
  collection: "posts",
  where: { status: { equals: "published" } },
  sort: "-publishedAt",
  limit: 10,
})
```

- Use the Local API for server-side data fetching (no HTTP overhead)
- Supports all CRUD operations with the same access control
- Use `depth` parameter to control relationship population depth
- Use `select` to fetch only needed fields

## Deployment

### Self-hosted
- Deploy as a Node.js application (Express, standalone)
- Use Postgres or MongoDB as the database
- Configure S3-compatible storage for uploads (AWS S3, Cloudflare R2, MinIO)
- Set up a reverse proxy (Nginx, Caddy) with TLS
- Run database migrations with `payload migrate`

### With Next.js
- Payload can run embedded in a Next.js application
- Use the Local API in Server Components for zero-overhead data fetching
- Share TypeScript types between Payload config and frontend components
- Deploy together on Vercel, Railway, or any Node.js host

## TypeScript integration

- Payload generates TypeScript types from the config automatically
- Types are generated at `payload-types.ts` (or configured output path)
- Regenerate types after every config change
- Use generated types for Local API queries and frontend components

## Review checklist

- Collections have appropriate access control for all operations
- Fields use correct types with validation rules
- Hooks handle side effects without breaking the save flow
- Rich text configured with custom blocks matching the design system
- Local API used for server-side data fetching
- TypeScript types generated and up to date
- Drafts and versions enabled for content with publish workflows
- Upload storage configured for production (S3-compatible)
- Database migrations version-controlled
- Admin panel customized for the editorial workflow

## Common issues to flag

- Missing access control on collections (data exposed to unauthorized users)
- Using REST API server-side when Local API is available (unnecessary overhead)
- Untyped queries (missing type generation)
- Rich text with default configuration (doesn't match design system)
- Hooks with unhandled errors (break the save flow)
- Missing draft/version support for content that needs editorial review
- Uploads stored on local filesystem in production (not persistent across deploys)
- Deep relationship population without `depth` limits (performance)
- Database migrations not version-controlled
- Missing field validations (editors can enter invalid data)
