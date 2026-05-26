---
name: journey-submit
description: Use as the final stage of the Butterbase journey when hackathon_mode is true and journey-deploy has passed. Assembles submission metadata (title, description, deployed URL, repo URL, app_id, team info) from the journey artifacts, confirms each field with the user one at a time, then calls prep_and_submit_hackathon_entry. Writes the receipt to docs/butterbase/05-submission.md.
---

# Journey: Hackathon submission

Stage 5 (final) of the guided journey. Submit to the active hackathon.

## When to use

- Dispatched by `journey` when `current_stage: submit` and `hackathon_mode: true`.
- Directly via `/butterbase:submit`.
- No-op (returns with `"submit is disabled outside hackathon mode"`) if `hackathon_mode: false`.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase:journey-preflight` first. Wait for it to return successfully before proceeding.

Additionally: refuse to run unless `deploy` is ticked in `00-state.md`. If it is not, tell the user to run `/butterbase:journey-deploy` first.

## Inputs

- `docs/butterbase/01-idea.md` — for title/tagline candidates.
- `docs/butterbase/02-plan.md` — for the feature list.
- `docs/butterbase/04-build-log.md` — for what actually shipped.
- `docs/butterbase/00-state.md` — for `app_id`, `deployed_url`.

## Procedure

Assemble each metadata field, then ask the user to confirm one at a time. Do not batch.

1. **Title.** Propose a candidate (≤60 chars) from `01-idea.md`. Ask: `"Title: '<candidate>' — keep / replace?"`.
2. **Short description.** Propose a one-sentence description from `01-idea.md` + must-haves. Ask: `"Description: '<candidate>' — keep / replace?"`.
3. **Deployed URL.** Use `deployed_url` from `00-state.md`. Ask: `"Deployed URL: <url> — correct?"`.
4. **Repo URL.** Ask: `"Public repo URL?"` (no default — required).
5. **App ID.** Use `app_id` from `00-state.md`. Ask: `"App ID: <id> — correct?"`.
6. **Team info.** Ask: `"Team / solo? Names + emails (one per line)?"`.
7. **Optional fields.** If the hackathon expects a demo video URL or screenshots: `"Demo video URL (or skip)?"`. Same for screenshots.

After all fields confirmed, show the assembled payload and ask: `"Submit now? (yes/no)"`. On yes, call `mcp__butterbase__prep_and_submit_hackathon_entry` with the assembled fields.

Capture the response (submission ID, timestamp) and write `docs/butterbase/05-submission.md`:

```markdown
# Submission

- submitted_at: <timestamp>
- submission_id: <id>
- title: <title>
- description: <desc>
- deployed_url: <url>
- repo_url: <url>
- app_id: <id>
- team: <names>
- demo_video: <url or n/a>
```

Tick `- [x] submit` in `00-state.md`, set `current_stage: done`. Print a one-line success message to the user.

## Outputs

- Hackathon submission via `prep_and_submit_hackathon_entry`.
- `docs/butterbase/05-submission.md`.

## Anti-patterns

- ❌ Calling `prep_and_submit_hackathon_entry` without explicit `yes` from the user.
- ❌ Inventing field values (especially repo URL or team emails) — always ask.
- ❌ Submitting before `journey-deploy` has passed.
