---
name: cms-migration
description: |
  CMS migration strategies — content mapping, data transformation, redirect handling, URL preservation, and zero-downtime cutover. Use when migrating between CMS platforms.
  Triggers: "migrate CMS", "move from WordPress", "switch CMS", "content migration", "migrate to Contentful", "redirect mapping", "move the content"
lastReviewed: 2026-04-10
upstreamDeps: []
---

# CMS Migration

Use this skill when planning or executing a migration between CMS platforms — content mapping, data transformation, redirect handling, URL preservation, SEO continuity, and zero-downtime cutover. Monogram regularly migrates clients between CMS platforms (WordPress → Contentful, Craft → Contentful, Swell → Shopify, etc.) as part of composable architecture projects.

## Core principles

1. **Content audit first** — Before writing any migration code, audit the source CMS to understand what content exists, what's actively used, and what can be archived or deleted. Migrating garbage is expensive.
2. **URL preservation is non-negotiable** — Every indexed URL must either be preserved or redirected. Losing URLs means losing SEO authority, inbound links, and traffic.
3. **Incremental over big-bang** — Migrate in phases when possible. Run the old and new systems in parallel, migrate section by section, and cut over gradually.
4. **Validate everything** — Automated validation after migration is essential. Compare page counts, content integrity, image references, internal links, and SEO metadata between source and target.

## Planning

### Content audit
- Inventory all content types, entry counts, and field structures in the source CMS
- Identify content that is actively used vs. archived/outdated
- Map source content types to target content types
- Document field-by-field mapping (source field → target field, with transformations)
- Identify content that needs manual review (complex layouts, custom widgets)

### URL mapping
- Export all published URLs from the source site
- Define the URL structure for the new site
- Create a redirect map: `old URL → new URL`
- Identify URLs that will remain the same (no redirect needed)
- Plan for query parameters, trailing slashes, and case sensitivity

### Timeline
- Phase 1: Content model setup in the target CMS
- Phase 2: Migration script development and testing
- Phase 3: Test migration (full run against staging)
- Phase 4: Content freeze in the source CMS
- Phase 5: Production migration
- Phase 6: Redirect deployment and validation
- Phase 7: DNS cutover (if changing domains)

## Migration scripts

### Architecture
- Write migration scripts in TypeScript for type safety
- Use the source CMS's export API and the target CMS's import API
- Process content in batches to avoid rate limits and memory issues
- Implement idempotent scripts (safe to re-run without duplicating content)
- Log every operation for debugging and audit

### Content transformation
- Transform rich text/HTML to the target CMS's format (Portable Text, Structured Text, Contentful Rich Text, etc.)
- Re-upload and re-reference media assets (images, files, videos)
- Preserve internal links by mapping source document IDs to target IDs
- Handle embedded content (videos, code blocks, custom widgets)
- Normalize data (trim whitespace, fix encoding, standardize date formats)

### Media migration
- Download all media assets from the source CMS
- Re-upload to the target CMS or asset storage (S3, CDN)
- Update all content references to point to the new asset URLs
- Verify image dimensions and formats are preserved
- Set appropriate alt text and metadata

### Reference resolution
- Build a mapping table: `source ID → target ID`
- Resolve content relationships after all content is migrated
- Handle circular references (migrate content first, then link)
- Validate that all references resolve correctly

## Redirects

### Implementation
- Implement redirects at the edge (Vercel `vercel.json`, Netlify `_redirects`, CDN rules)
- Use `301` for permanent redirects (SEO authority transfer)
- Use `308` for permanent redirects that must preserve the HTTP method
- Keep redirect chains to a single hop (never chain redirects)
- Test redirects before DNS cutover

### Common patterns
```json
{
  "redirects": [
    { "source": "/blog/:slug", "destination": "/articles/:slug", "permanent": true },
    { "source": "/category/:cat/:slug", "destination": "/blog/:slug", "permanent": true },
    { "source": "/old-page", "destination": "/new-page", "permanent": true }
  ]
}
```

### Monitoring
- Monitor 404 errors after launch to catch missing redirects
- Use Google Search Console to track indexing status
- Set up alerts for spikes in 404 errors
- Keep the redirect map as a living document — add new redirects as needed

## SEO continuity

### Pre-migration
- Export all page titles, meta descriptions, and canonical URLs from the source
- Document structured data (JSON-LD) on key pages
- Record current search rankings for important keywords
- Submit the current sitemap to Google Search Console

### Post-migration
- Verify all page titles and meta descriptions are preserved
- Verify structured data is present and valid
- Submit the new sitemap to Google Search Console
- Request re-indexing of key pages
- Monitor search rankings for 4-6 weeks post-migration

## Validation

### Automated checks
- Compare page counts between source and target
- Verify all URLs are either preserved or redirected
- Check that all images load correctly
- Validate internal links (no broken links)
- Compare SEO metadata (titles, descriptions, canonicals)
- Verify structured data with Google's Rich Results Test

### Manual review
- Spot-check representative pages from each content type
- Verify rich text formatting is preserved
- Check responsive layouts on mobile and desktop
- Verify forms, interactive elements, and third-party embeds
- Test the editorial workflow in the new CMS

## Rollback plan

- Keep the source CMS running until the migration is validated
- Maintain DNS records for quick rollback
- Document the rollback procedure before starting
- Set a go/no-go deadline (e.g. 48 hours after cutover)
- After validation, decommission the source CMS and archive the data

## Review checklist

- Content audit completed with field-by-field mapping
- URL redirect map covers all published URLs
- Migration scripts are idempotent and logged
- Media assets migrated and references updated
- Rich text transformed to the target CMS format
- Internal links resolved with the ID mapping table
- Redirects implemented and tested
- SEO metadata preserved (titles, descriptions, canonicals, structured data)
- Sitemap submitted to Google Search Console
- 404 monitoring configured post-launch
- Rollback plan documented and tested

## Common issues to flag

- Missing redirect map (URLs return 404 after migration)
- Rich text formatting lost during transformation
- Broken internal links (source IDs not mapped to target IDs)
- Media assets not migrated (broken images)
- SEO metadata not preserved (titles, descriptions reset to defaults)
- Non-idempotent migration scripts (duplicate content on re-run)
- No content freeze during production migration (content drift)
- Redirect chains (old URL → intermediate URL → new URL)
- Missing validation step (content integrity not verified)
- No rollback plan (can't revert if migration fails)
