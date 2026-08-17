# Monogram Skills

Agent skills from Monogram Labs, installable in Claude Code, Cursor, and any agent supported by the [skills CLI](https://github.com/vercel-labs/skills).

## Install

```bash
npx skills add monogram-labs/skills
```

The CLI lets you choose which skills to install, and where (project or global).

## Skills

| Skill | Description |
| ----- | ----------- |
| [elevenlabs-demo-video](skills/elevenlabs-demo-video) | Build a narrated product-demo video of a running web app: Playwright screen recording, ElevenLabs voiceover, ffmpeg music bed. |
| [harvest-entry](skills/harvest-entry) | Turn the day's git commits into a categorized Harvest time-entry note: standup notes, daily logs, and work reports. |
| [monogram-voice](skills/monogram-voice) | Ghostwrite content in Monogram's brand voice: case studies, technical and AEO blog posts, comparison guides, and announcements that read like monogram.io. |

### monogram-voice

The bundled spec (`monogram-voice.md`) was distilled from all 84 live pages of monogram.io and adversarially refined against real site copy. It covers identity and stance, voice and punctuation rules by register, per-content-type playbooks, verbatim "sounds like us" examples, anti-slop guardrails, and the site's fixed page chrome: CTA and footer blocks, byline formats, and Scope tokens. The skill has the agent draft against the spec and run its final checklist before returning a piece.

## Adding a skill

Create a directory under `skills/` containing a `SKILL.md` with `name` and `description` frontmatter, plus any reference files the skill needs. The skills CLI discovers every skill directory under `skills/`.

## License

[MIT](LICENSE)
