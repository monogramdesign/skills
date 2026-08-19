# Monogram Skills

Agent skills from Monogram Labs, installable in Claude Code, Cursor, and any agent supported by the [skills CLI](https://github.com/vercel-labs/skills).

## Install

```bash
npx skills add monogramdesign/skills
```

The CLI lets you choose which skills to install, and where (project or global).

## Skills

| Skill | Description |
| ----- | ----------- |
| [cms-migration](skills/cms-migration) | CMS migration strategies: content mapping, data transformation, redirect handling, URL preservation, and zero-downtime cutover between platforms. |
| [contentful](skills/contentful) | Contentful CMS: content types, rich text rendering, environments, migrations, webhook revalidation, and TypeScript integration. |
| [datocms](skills/datocms) | DatoCMS: structured text, modular content, GraphQL API, real-time previews, image optimization, and TypeScript integration. |
| [elevenlabs-demo-video](skills/elevenlabs-demo-video) | Build a narrated product-demo video of a running web app: Playwright screen recording, ElevenLabs voiceover, ffmpeg music bed. |
| [harvest-entry](skills/harvest-entry) | Turn the day's git commits into a categorized Harvest time-entry note: standup notes, daily logs, and work reports. |
| [monogram-voice](skills/monogram-voice) | Ghostwrite content in Monogram's brand voice: case studies, technical and AEO blog posts, comparison guides, and announcements that read like monogram.io. |
| [payload](skills/payload) | Payload CMS: collections, globals, access control, hooks, rich text, self-hosted deployment, and TypeScript integration. |
| [prismic](skills/prismic) | Prismic CMS: Slice Machine, content modeling, rich text, previews, webhook revalidation, and TypeScript integration. |
| [sanity](skills/sanity) | Sanity CMS: GROQ queries, Studio customization, structured content, real-time previews, Visual Editing, and TypeScript integration. |
| [seo](skills/seo) | Technical SEO for Next.js: metadata, structured data, sitemaps, Core Web Vitals, crawlability, and rendering strategies. |
| [shopify](skills/shopify) | Headless Shopify Plus: Storefront API, checkout, subscriptions, webhooks, and product catalog management. |
| [spec-driven-development](skills/spec-driven-development) | Spec-driven development: writing specifications before code, multi-file spec format, task generation, delta specs, and lifecycle management. |
| [ux-copywriter](skills/ux-copywriter) | Microcopy, error messages, onboarding flows, CTAs, empty states, tooltips, and tone consistency for user-facing text. |
| [web-performance](skills/web-performance) | Core Web Vitals optimization, resource loading, image and font optimization, caching, bundle size reduction, and runtime JavaScript performance. |

### monogram-voice

The bundled spec (`monogram-voice.md`) was distilled from all 84 live pages of monogram.io and adversarially refined against real site copy. It covers identity and stance, voice and punctuation rules by register, per-content-type playbooks, verbatim "sounds like us" examples, anti-slop guardrails, and the site's fixed page chrome: CTA and footer blocks, byline formats, and Scope tokens. The skill has the agent draft against the spec and run its final checklist before returning a piece.

## Adding a skill

Create a directory under `skills/` containing a `SKILL.md` with `name` and `description` frontmatter, plus any reference files the skill needs. The skills CLI discovers every skill directory under `skills/`.

## License

[MIT](LICENSE)
