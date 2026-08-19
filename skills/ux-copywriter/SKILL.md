---
name: ux-copywriter
description: |
  Microcopy, error messages, onboarding flows, CTAs, empty states, tooltips, and tone consistency. Use when writing or reviewing user-facing text.
  Triggers: "write copy", "error message", "button text", "empty state", "UI text", "what should this say", "onboarding copy", "CTA wording"
lastReviewed: 2026-04-10
upstreamDeps: []
---

# UX Copywriter

Use this skill when writing or reviewing user-facing text in web applications — microcopy, error messages, onboarding flows, CTAs, empty states, tooltips, and tone consistency. Treat every word in the UI as a design decision: make interfaces clear, helpful, and human.

## Core principles

1. **Clarity over personality** — Users are trying to accomplish a task, not read prose. Be clear first; avoid cleverness that obscures meaning.
2. **Front-load the action** — Put the most important information or action first. Users scan; they do not read every word.
3. **Be specific** — "Something went wrong" helps no one. "Your image couldn't be uploaded because it exceeds 10MB" helps everyone.
4. **Consistent voice** — Use the same words for the same concept everywhere. Do not call it "workspace" in one place and "project" in another.

## Microcopy patterns

### Buttons and CTAs
- Use verbs that describe the action: "Save changes", "Send invite", "Delete workspace"
- Avoid generic labels: "Submit", "OK", "Click here", "Continue"
- For destructive actions, use explicit language: "Delete 3 images permanently"
- Make the primary action visually prominent and secondary actions subdued

### Error messages
- State what happened, why, and what to do next
- Use plain language (not error codes or technical jargon)
- Never expose internal details (stack traces, database errors, file paths)
- Offer a recovery path when possible

```
Bad:  "Error 500: Internal Server Error"
Good: "We couldn't save your changes. Check your connection and try again."

Bad:  "Invalid input"
Good: "Email address must include an @ symbol"
```

### Empty states
- Explain what will appear here once the user takes action
- Include a CTA to get started
- Use illustration or icon so the space feels intentional, not broken

### Loading states
- Tell the user what's happening: "Loading your images..." not just a spinner
- For long operations, show progress: "Uploading 3 of 12 images..."
- Avoid "Please wait" — it implies the system is slow

### Confirmation dialogs
- Title states the action: "Delete workspace?"
- Body explains consequences: "This will permanently delete all images and settings. This cannot be undone."
- Match the confirm button to the action: "Delete workspace" (not "OK" or "Yes")
- Keep Cancel always available and clearly labeled

### Tooltips and help text
- Explain why, not what (the label already says what)
- Keep to one sentence
- Use for non-obvious features, not for every field

### Onboarding
- Teach one concept per step
- Show value before asking for effort
- Let users skip and come back later
- Use progressive disclosure (do not overwhelm with options)

### Success messages
- Confirm the action was completed: "Invite sent to user@example.com"
- Include next steps when relevant: "They'll receive an email with instructions"
- Auto-dismiss after a few seconds (do not require interaction)

## Voice and tone

- **Professional but human** — not corporate, not casual
- **Confident** — "Your changes are saved" not "Your changes should be saved"
- **Respectful of time** — short sentences, no filler words
- **Inclusive** — avoid jargon, idioms, and cultural assumptions
- **Consistent** — same terms for same concepts throughout the app

## Review checklist

- Button labels describe the specific action
- Error messages explain what happened and what to do
- Empty states include a CTA
- Destructive actions have explicit confirmation with consequences
- Loading states describe what's happening
- Terminology is consistent across the app
- No technical jargon in user-facing text
- No internal details exposed in error messages
- Tone is consistent throughout

## Common issues to flag

- Generic button labels ("Submit", "OK", "Click here")
- Error messages that expose internal details
- Empty states with no guidance or CTA
- Inconsistent terminology (mixing "workspace" and "project")
- Loading states with no description (just a spinner)
- Confirmation dialogs with "Yes"/"No" instead of action-specific labels
- Passive voice ("An error was encountered" → "We couldn't save your changes")
- Unnecessary words ("Please click the button below to submit" → "Save changes")
