---
name: spec-driven-development
description: |
  Spec-driven development — writing specifications before code, multi-file spec format, task generation, delta specs, lifecycle management, and project setup. Use when setting up SDD in a project, writing feature specs, or improving an existing spec workflow.
  Triggers: "write a spec", "spec-driven", "specification", "requirements doc", "feature spec", "set up specs", "spec format", "spec workflow", "spec template", "delta spec", "spec tasks"
metadata:
  category: code/skills
lastReviewed: 2026-04-16
upstreamDeps: []
---

# Spec-Driven Development

Use this skill when adopting spec-driven development (SDD) in a project, writing feature specifications, or improving an existing spec workflow. SDD means every meaningful change starts as a written specification that is reviewed and approved before implementation begins.

## When to use

- Setting up SDD in a new or existing project
- Writing a feature, migration, or integration spec
- Reviewing whether a spec is ready for implementation
- Checking if an implementation matches its spec
- Generating implementation tasks from an approved spec
- Evolving a spec mid-implementation with delta specs
- Improving spec quality or process maturity

## Why specs

- **Alignment** — forces the author to think through requirements, edge cases, and trade-offs before writing code.
- **Review leverage** — reviewers catch design problems when they're cheap to fix (before code exists).
- **Scope control** — a written spec makes scope creep visible. Anything not in the spec is out of scope.
- **Onboarding** — new contributors read the spec to understand *why* the code exists, not just *what* it does.
- **Testability** — acceptance criteria in the spec become the test plan.
- **Task generation** — a well-structured spec can be decomposed into discrete, ordered implementation tasks that agents execute methodically.

## Project setup

### Directory structure

Specs live in a `specs/` directory at the project root. Each spec is a folder containing up to four markdown files.

```
specs/
├── _templates/               # starter files for new specs
│   ├── requirements.md
│   ├── design.md
│   └── tasks.md
├── auth-magic-link/
│   ├── requirements.md       # what to build and why
│   ├── design.md             # how to build it
│   └── tasks.md              # ordered implementation steps
├── cart-checkout-flow/
│   ├── requirements.md
│   ├── design.md
│   ├── tasks.md
│   └── delta-001.md          # requirement change during implementation
└── search-indexing/
    ├── requirements.md
    └── design.md             # tasks.md not yet generated
```

### Naming

- Use kebab-case for spec folders: `feature-name/`
- Be specific: `auth-magic-link/` not `auth/`
- One spec per feature/change — don't combine unrelated work
- Delta specs are numbered sequentially: `delta-001.md`, `delta-002.md`

## Spec format — multi-file structure

Each spec is a folder with three core files. This separation lets agents load only what they need: requirements during review, design during architecture decisions, tasks during implementation.

### File 1: `requirements.md`

Defines *what* to build and *why*. This is the file that gets reviewed and approved.

#### Frontmatter (required)

```yaml
---
title: Magic link authentication
status: draft
author: @github-handle
created: 2026-04-16
updated: 2026-04-16
---
```

**Status lifecycle:**

| Status | Meaning |
|--------|---------|
| `draft` | Being written, not ready for review |
| `review` | Ready for team review |
| `approved` | Reviewed and approved, ready to implement |
| `implementing` | Implementation in progress |
| `implemented` | Code is merged and matches the spec |
| `superseded` | Replaced by another spec (link to it) |
| `abandoned` | Will not be implemented (document why) |

#### Sections

**Overview** (required) — 1-3 sentences: what this spec covers and why. A reader should know within 10 seconds whether it's relevant.

**Background** — context that isn't obvious: prior art, constraints from external systems, business drivers. Skip if the overview is sufficient.

**Requirements** (required) — numbered list of concrete, testable requirements. Each should be independently verifiable.

Good requirements:
- "The system sends a magic link email within 5 seconds of form submission."
- "Expired links (older than 15 minutes) redirect to the sign-in page with an error message."

Bad requirements:
- "The auth system should be fast." (not testable)
- "Handle errors properly." (not specific)

Group under subheadings when there are more than 7-8.

**Non-goals** — explicitly state what this spec does *not* cover. Prevents scope creep and avoids misunderstandings during review.

**Edge cases** (required) — enumerate edge cases and state how each is handled. Format: "When [condition], then [expected behavior]."

**Acceptance criteria** (required) — a checklist that must pass for the spec to be considered implemented. These map directly to test cases.

```markdown
- [ ] Magic link email arrives within 5 seconds
- [ ] Clicking an expired link shows "Link expired, request a new one"
- [ ] Clicking a used link shows "Link already used, sign in again"
- [ ] User is redirected to their original destination after sign-in
```

**Dependencies** — external systems, third-party services, or other specs this work depends on.

**Open questions** — unresolved decisions that need team input. Every open question must be resolved before status moves to `approved`.

### File 2: `design.md`

Documents *how* the requirements will be met. Created after requirements are approved (or in parallel for design-first workflows).

#### Frontmatter

```yaml
---
title: Magic link authentication — Design
parent: auth-magic-link
---
```

#### Sections

**Architecture** — which layers/modules are affected, data flow. Use Mermaid diagrams when they clarify relationships.

**API surface** — new endpoints, changed interfaces, request/response shapes. Use TypeScript type definitions or JSON examples.

**Data model** — schema changes, new entities, migrations. Be explicit about field names, types, and constraints.

**UI changes** — new screens, modified components, interaction patterns.

**Error handling** — how each error class is surfaced to the user or logged.

**Testing strategy** — which requirements need unit tests, integration tests, or e2e tests.

**Migration / rollback plan** — for specs that change existing behavior: how to migrate data, feature-flag the rollout, and roll back if something goes wrong.

Avoid writing implementation-level code. The design describes *what* the system does at the boundary level, not *how* every function works internally.

### File 3: `tasks.md`

Ordered implementation tasks derived from the design. Each task is discrete, independently verifiable, and small enough to complete in one session.

#### Frontmatter

```yaml
---
title: Magic link authentication — Tasks
parent: auth-magic-link
generated: 2026-04-16
linear:
  team: ENG
  project: auth-magic-link
---
```

The `linear` field is optional. When present, tasks are synced to Linear as issues. When absent, tasks live only in the markdown file. See the **Linear integration** section below.

#### Task format

```markdown
## Tasks

- [ ] **T1: Create magic link token schema**
  Add `magic_link_tokens` table with columns: id, user_email, token_hash,
  expires_at, used_at, created_at. Add migration file.
  _Verifies: R1, R2_

- [ ] **T2: Implement token generation endpoint**
  POST /api/auth/magic-link accepts { email }. Generates token, stores hash,
  sends email via SendGrid. Returns 200 with { sent: true }.
  _Verifies: R1, R3_

- [x] **T3: Implement token verification endpoint**
  GET /api/auth/verify?token=xxx. Validates token exists, not expired, not used.
  Marks as used. Creates session. Redirects to stored destination or /.
  _Verifies: R2, R4, R5_

- [ ] **T4: Add sign-in UI with email form**
  Create /sign-in page with email input. On submit, call token generation
  endpoint. Show "Check your email" confirmation.
  _Verifies: R1_
```

#### Task conventions

- Tasks are ordered by dependency — earlier tasks unblock later ones
- Each task references which requirements it verifies (`_Verifies: R1, R3_`)
- Tasks use checkboxes for status: `- [ ]` pending, `- [x]` complete
- Task IDs are sequential: T1, T2, T3…
- Each task describes *what* to do and *what to verify*, not step-by-step code
- A task should be completable in a single focused session (30-90 minutes of work)
- When synced to Linear, each task includes a `_Linear: ENG-123_` annotation after the verifies line

#### Task format with Linear

When tasks are synced to Linear, each task carries its issue identifier:

```markdown
- [ ] **T1: Create magic link token schema**
  Add `magic_link_tokens` table with columns: id, user_email, token_hash,
  expires_at, used_at, created_at. Add migration file.
  _Verifies: R1, R2_
  _Linear: ENG-42_

- [x] **T2: Implement token generation endpoint**
  POST /api/auth/magic-link accepts { email }. Generates token, stores hash,
  sends email via SendGrid. Returns 200 with { sent: true }.
  _Verifies: R1, R3_
  _Linear: ENG-43_
```

The `_Linear: ENG-43_` annotation is the source of truth for linking back to the Linear issue. Status flows bidirectionally: checking a box in `tasks.md` should be reflected in Linear, and completing an issue in Linear should be reflected in `tasks.md`.

## Linear integration

Tasks can optionally be tracked in Linear alongside the `tasks.md` file. This is opt-in — the entire spec system works without Linear. When Linear is available, it adds visibility for project managers and team leads without changing the developer workflow.

### When to use Linear

- The team already uses Linear for project tracking
- Stakeholders need visibility into spec progress without reading markdown
- Multiple people are working on tasks from the same spec
- The project requires formal status reporting

### When to skip Linear

- Solo work or prototyping
- Linear is not set up for the project
- The spec has fewer than 3-4 tasks (overhead isn't worth it)

### How it works

1. **Generate tasks locally first** — `tasks.md` is always created first via the **generate-spec-tasks** command. It is the source of truth for task content and requirement tracing.

2. **Sync to Linear on request** — when the user asks to sync, each task becomes a Linear issue in the specified team/project. The `_Linear: ENG-123_` annotation is added to `tasks.md`.

3. **Bidirectional status** — task completion can be tracked in either place:
   - Checking a box in `tasks.md` → agent updates the Linear issue status
   - Completing an issue in Linear → agent updates the checkbox in `tasks.md`

4. **Linear is secondary** — if Linear is unavailable or the MCP is not connected, the system falls back to markdown-only tracking. No functionality is lost.

### `tasks.md` frontmatter with Linear

```yaml
---
title: Magic link authentication — Tasks
parent: auth-magic-link
generated: 2026-04-16
linear:
  team: ENG
  project: auth-magic-link
---
```

| Field | Required | Description |
|-------|----------|-------------|
| `linear.team` | Yes (if syncing) | Linear team key to create issues in |
| `linear.project` | No | Linear project to group issues under. Omit to create standalone issues. |

### Sync conventions

- Task title becomes the Linear issue title (without the T-number prefix)
- Task description becomes the issue description, with a footer linking back to the spec: `Spec: specs/<feature>/requirements.md | Verifies: R1, R3`
- Labels: add a `spec` label to all synced issues for filtering
- The `_Linear: ENG-123_` annotation in `tasks.md` is the bidirectional link — never remove it manually
- When a delta spec adds or removes tasks, the corresponding Linear issues are created or cancelled

### Status mapping

| `tasks.md` | Linear status |
|------------|---------------|
| `- [ ]` (unchecked) | Todo / Backlog |
| In progress (agent working) | In Progress |
| `- [x]` (checked) | Done |
| Task removed by delta | Cancelled |

## Delta specs

Delta specs handle requirement changes during implementation. Instead of silently editing `requirements.md` (which loses change history), create a delta file that explicitly communicates what changed.

### When to use delta specs

- A requirement needs to change after status is `approved` or `implementing`
- New requirements emerge during implementation
- A requirement is being removed or descoped

### Delta file format: `delta-NNN.md`

```yaml
---
title: Magic link authentication — Delta 001
parent: auth-magic-link
status: proposed
author: @github-handle
created: 2026-04-16
---
```

**Status lifecycle for deltas:**

| Status | Meaning |
|--------|---------|
| `proposed` | Change proposed, needs review |
| `approved` | Reviewed and accepted |
| `applied` | Changes merged into requirements.md |
| `rejected` | Change was not accepted (document why) |

### Delta body format

Use ADDED, MODIFIED, and REMOVED markers to describe changes relative to the current `requirements.md`:

```markdown
## Reason

[1-2 sentences: why this change is needed. What was learned during
implementation that prompted it.]

## Changes

### ADDED

**R9: Rate limiting on magic link requests**
The system limits magic link requests to 3 per email address per 15-minute
window. Exceeding the limit returns a 429 response with a retry-after header.

_Acceptance criteria:_
- [ ] Fourth request within 15 minutes returns 429
- [ ] Response includes Retry-After header with seconds remaining

### MODIFIED

**R2: Token expiration** (was: 15 minutes → now: 30 minutes)

Before: "Expired links (older than 15 minutes) redirect to the sign-in page."
After: "Expired links (older than 30 minutes) redirect to the sign-in page."

Reason: User research showed 15 minutes is too short for users who check email
on a different device.

_Updated acceptance criteria:_
- [ ] Links older than 30 minutes show expiration message

### REMOVED

**R7: Magic link SMS fallback**

Reason: SMS provider integration is descoped to a future spec. Removes
dependency on Twilio and simplifies initial launch.

## Impact on tasks

- T2 needs updating: add rate limiting check before token generation
- New task needed: T7 for rate limiting middleware
- T6 (SMS fallback) should be removed
```

### Applying deltas

Once a delta is `approved`:

1. Update `requirements.md` with the changes
2. Update `tasks.md` to reflect new/modified/removed tasks
3. Set the delta's status to `applied`
4. Update `requirements.md` frontmatter `updated` date

Deltas are kept in the spec folder as a change history record even after being applied.

## Templates

### `_templates/requirements.md`

```markdown
---
title: [Feature name]
status: draft
author: @[github-handle]
created: [YYYY-MM-DD]
updated: [YYYY-MM-DD]
---

## Overview

[1-3 sentences: what and why]

## Background

[Context, prior art, constraints — skip if overview is enough]

## Requirements

1. [Concrete, testable requirement]
2. [Another requirement]

## Non-goals

- [What this spec explicitly does not cover]

## Edge cases

- When [condition], then [expected behavior].

## Acceptance criteria

- [ ] [Criterion that maps to a test case]

## Dependencies

- [External system, service, or spec this depends on]

## Open questions

- [ ] [Unresolved decision that needs team input]
```

### `_templates/design.md`

```markdown
---
title: [Feature name] — Design
parent: [spec-folder-name]
---

## Architecture

[Affected layers, data flow, module boundaries]

## API surface

[New/changed endpoints, interfaces, request/response shapes]

## Data model

[Schema changes, new entities, migrations]

## UI changes

[New screens, modified components, interaction patterns]

## Error handling

[How each error class is surfaced or logged]

## Testing strategy

[Which requirements need unit/integration/e2e tests]

## Migration / rollback plan

[Data migration, feature flags, rollback procedure — skip for greenfield]
```

### `_templates/tasks.md`

```markdown
---
title: [Feature name] — Tasks
parent: [spec-folder-name]
generated: [YYYY-MM-DD]
---

## Tasks

- [ ] **T1: [Short task title]**
  [What to do and what to verify. 2-3 sentences max.]
  _Verifies: R1_

- [ ] **T2: [Short task title]**
  [What to do and what to verify.]
  _Verifies: R2, R3_
```

## Spec quality checklist

Use this when reviewing a spec before approving it.

### Completeness
- All three sections present in `requirements.md`: requirements, edge cases, acceptance criteria
- Requirements are numbered and testable
- Edge cases cover error states, empty inputs, concurrency, and boundary values
- Acceptance criteria are specific enough to write tests from
- Open questions are all resolved (for `approved` status)

### Clarity
- Overview is understandable without reading the rest
- No ambiguous pronouns ("it should handle this" — what is "it"? what is "this"?)
- Technical terms are used consistently
- Requirements don't overlap or contradict each other

### Scope
- Non-goals section exists and is meaningful
- No requirement is actually a separate feature that deserves its own spec
- Design doesn't prescribe internal implementation details unnecessarily

### Testability
- Every requirement has a corresponding acceptance criterion
- Acceptance criteria are pass/fail, not subjective
- Edge cases state both the condition and the expected outcome

### Feasibility
- Dependencies are identified and available
- No requirement assumes capabilities that don't exist yet (unless noted in dependencies)
- Migration/rollback plan exists for changes to existing behavior

### Task quality (when tasks.md exists)
- Every task references at least one requirement
- Every requirement is covered by at least one task
- Tasks are ordered by dependency — no task references work from a later task
- Each task is completable in one focused session
- Task descriptions state what to verify, not just what to do

## Process maturity levels

### Level 1 — Ad hoc
- Specs written for some features, not all
- No consistent format
- Review is optional

### Level 2 — Consistent
- All features above a size threshold get a spec
- Standard multi-file template used
- Specs are reviewed before implementation starts

### Level 3 — Integrated
- Specs are the source of truth for implementation scope
- Tasks are generated from specs and drive implementation
- Acceptance criteria drive the test plan
- PR descriptions reference the spec and note any deviations
- Spec status is updated as implementation progresses
- Delta specs track requirement changes

### Level 4 — Continuous
- Specs are updated when requirements change mid-implementation (via deltas)
- Drift between spec and code is caught in review
- Task completion is tracked and reported
- Retrospectives include spec quality as a dimension
- New team members write their first spec within the first week

## Common anti-patterns

- **Spec-after-the-fact** — writing the spec to document code that already exists. The spec should *precede* the code.
- **Novel-length specs** — if the spec is longer than the implementation will be, it's too detailed. Describe *what*, not *how*.
- **Rubber-stamp review** — approving specs without reading them defeats the purpose. Review specs as carefully as code.
- **Perpetual draft** — specs that stay in `draft` indefinitely. Set a time limit: if it's not in `review` within a week, either finish it or abandon it.
- **Ghost specs** — approved specs that are never implemented and never abandoned. Clean up quarterly.
- **Scope anchoring** — refusing to update a spec when new information emerges during implementation. Use delta specs instead.
- **Silent edits** — editing `requirements.md` directly after approval without a delta. Loses change history and skips review.
- **Monolith tasks** — tasks that take days or bundle multiple requirements. Break them down until each is a single session.
