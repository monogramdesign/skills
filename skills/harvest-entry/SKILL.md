---
name: harvest-entry
description: Write the notes field for a Harvest time entry — a short, categorized bulleted summary of the day's work, drawn only from the user's own git commits. Use this whenever the user mentions Harvest, logging hours, time tracking, or timesheets, and whenever they ask what they did today, ask for a summary of their changes, or need standup notes, a daily log, or a work report — even if they never say the word "commits."
---

# Harvest Entry

Turn today's commits into the notes field of a Harvest time entry: a categorized
summary the user pastes and submits. The reader is a project manager or client,
so the altitude is product surfaces and user-facing behavior — not files,
symbols, or diffs.

## 1. Gather (the user's own commits, nothing else)

```bash
git log --all --no-merges --author="$(git config user.email)" \
  --since=midnight --pretty=format:'%h%n%s%n%b%n---' --stat
```

If it isn't in one of these commits, it didn't happen — never read the working
tree, staged changes, stashes, or other people's commits. Billable hours should
not rest on someone else's work or on an experiment that never landed. If the
log is empty but `git log --since=midnight` isn't, the user commits under
another identity; try `git config user.name`. For "yesterday" or "this week,"
swap the date flags and change nothing else.

Commit subjects lie by omission. `style: polish`, `fix: cleanup`, and
`chore: tweaks` tell you nothing, and a day of them will collapse into a
meaningless "Other changes" group. When a subject is vague, run
`git show <hash>` and read the actual diff.

## 2. Find the themes

Cluster commits by theme *before* writing anything. A theme is a coherent body
of work, and it comes in three shapes:

- **A project** — several commits converging on one deliverable. Usually
  obvious from the subjects.
- **A surface** — repeated work on one area: onboarding, billing, the reports
  page.
- **A cross-cutting concern** — the same *kind* of change scattered across many
  areas. These hide, because no two commits look alike. Look for the tell in
  the diffs, not the subjects:
  - *responsive / mobile*: breakpoint prefixes (`sm:`, `md:`), media queries,
    viewport units, tap-target sizing, overflow and stacking fixes on narrow
    screens
  - *accessibility*: aria attributes, focus rings, keyboard handlers, contrast
  - *design system*: tokens replacing literals, spacing and radius
    normalization, icon and typography unification
  - *performance*: memoization, query batching, bundle splitting, lazy loading
  - *types / tests / errors*: added coverage, narrowed types, error boundaries

Three or more commits sharing a cross-cutting concern have earned their own
group. Give it a real heading — `Polished mobile layouts across the app:` — and
never let it dissolve into the catch-all. That dissolution is the most common
way this entry goes wrong: a real afternoon of billable work disappears into one
vague bullet because the commits touched a dozen unrelated files.

The catch-all group holds genuine miscellany only: one-off changes that share
nothing with each other.

## 3. Write it up

Plain text. Each group is a line ending in a colon, then hyphen bullets. No
markdown headers, no bold, no hashes — Harvest's notes field renders none of it.

Headings are past-tense verb phrases naming a deliverable — `Shipped the first
pass of the notification center:`, `Polished mobile layouts across the app:`.
They read like line items on an invoice: work already done. The catch-all comes
last and is a noun phrase starting with "Other."

Bullets open with a past-tense verb, run 8–20 words (25 is the hard ceiling),
and merge related commits rather than transcribing them — five commits fixing
one flaky test are one bullet. Name product surfaces and user-facing concepts
freely (the overview page, the sidebar, Gmail handoff, Tailwind classes); never
name files, paths, functions, or classes. A semicolon can join two tightly
related actions; a parenthetical can carry scope or alternatives considered. A
bullet that overruns is describing implementation — raise the altitude and it
shrinks.

Cap at 3 groups and 4 bullets per group. These are ceilings, not targets: a
quiet day is one group with two bullets. But when the day overflows the cap,
**merge bullets — never drop work.** Dropping is how the mobile polish vanishes.

## 4. Return the entry and nothing else

The first character of the response is the first group heading; the last is the
end of the final bullet. Nothing wraps it. In particular, do not report:

- how many commits there were, or that they were all the user's
- the user's name or email, or the branch, or the date range
- a closing sentence characterizing the day ("centered on a reveal-animation
  pass with surrounding cleanup")
- a preamble ("Here's your summary:") or an offer to adjust it

This text goes straight into a Harvest notes field. Any sentence that isn't a
heading or a bullet has to be deleted by hand before it can be pasted. Scoping
and verification are your business, not the reader's — if a check failed, fix it
silently rather than narrating it.

The only exception: if the user has no commits today, say so in one line. That
is the whole response.

## Example

Commits:

```
a1b2c3d  feat(notifications): bell icon + dropdown shell in TopNav
e4f5g6h  feat(notifications): group activity feed by entity type
i7j8k9l  feat(notifications): per-type mute toggles, mark-all-read
q3r4s5t  docs: RFC for in-app vs email digest delivery
b1c2d3e  fix: sidebar overflows viewport under 480px
f4g5h6i  style: stack overview cards on narrow screens
j7k8l9m  fix: filter chip tap targets below 44px
n0o1p2q  style: reduce header padding at sm breakpoint
u6v7w8x  feat(reports): multi-select owner filter
o1p2q3r  refactor: replace inline styles with design tokens
```

Entry:

```
Shipped the first pass of the notification center:
- Scoped a notification center concept and delivery preferences (in-app vs. email digest)
- Built the end-to-end flow: bell entry point, grouped activity feed, mute controls, mark-all-read

Polished mobile layouts across the app:
- Fixed sidebar overflow and undersized tap targets on narrow viewports
- Reflowed the overview cards and tightened header spacing at small breakpoints

Other functionality and cleanup:
- Added multi-select owner filtering to the reports page
- Migrated remaining inline styles to design tokens
```

Every line is past tense — headings included. The response is that block and
nothing more: it starts at `Shipped` and ends at `tokens`. (The next paragraph
is a note to you, not part of any output.)

Four scattered commits — two `fix`, two `style`, no shared files — became a
named group because the diffs were all about small screens. Without reading
them, they'd have been "Other styling changes" or nothing at all.

## Before returning

Every bullet: under 25 words, past-tense verb, no file or symbol names, purpose
over mechanics. Every heading: past tense too, except the "Other" catch-all.
Every group: earns its heading, catch-all last, nothing dropped to fit the caps.
Then delete every line that is not a heading or a bullet.
