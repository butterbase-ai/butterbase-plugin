---
name: journey-submit
description: Use as the final stage of the Butterbase journey when hackathon_mode is true and journey-deploy has passed. Resolves which hackathon to submit to (asking the user when multiple are open), walks every field in the hackathon's returned field_schema with the user one at a time, then calls prep_and_submit_hackathon_entry. Writes the receipt to docs/butterbase/05-submission.md.
---

# Journey: Hackathon submission

Stage 5 (final) of the guided journey. Resolve the active hackathon, confirm fields, submit.

## When to use

- Dispatched by `journey` when `current_stage: submit` and `hackathon_mode: true`.
- Directly via `/butterbase-skills:submit`.
- No-op (returns with `"submit is disabled outside hackathon mode"`) if `hackathon_mode: false`.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

Additionally: refuse to run unless `deploy` is ticked in `00-state.md`. If it is not, tell the user to run `/butterbase-skills:journey-deploy` first.

## Inputs

- `docs/butterbase/01-idea.md` — title / tagline candidates.
- `docs/butterbase/02-plan.md` — feature list.
- `docs/butterbase/04-build-log.md` — what actually shipped.
- `docs/butterbase/00-state.md` — `app_id`, `deployed_url`.

## Procedure

### Step 1 — Prep (resolve which hackathon)

Always start with `action: "prep"`. Do **not** assume there is exactly one hackathon, and do **not** invent a field schema from prior context — the platform owns the schema and it varies per hackathon.

```jsonc
// First, ask the user: "Do you have a submission code from the organizer? (paste it, or 'no')"
// If yes, pass it. If no, omit it.
{
  "tool": "prep_and_submit_hackathon_entry",
  "arguments": {
    "action": "prep",
    "submission_code": "<optional, from user>"
  }
}
```

The response shape is:

```jsonc
{
  "matched": null | { "slug", "name", "submission_deadline", "ends_at", "field_schema": { "fields": [...] } },
  "match_reason": null | "submission_code" | "already_bound" | "single_open",
  "open_hackathons": [{ "slug", "name", "starts_at", "ends_at", "submission_deadline" }, ...],
  "next_call": { ... }   // present only when matched is non-null
}
```

Branch on the result:

| `matched` | `open_hackathons.length` | Action |
|---|---|---|
| non-null | any | Resolved — continue to Step 2 with `matched.field_schema`. Mention `match_reason` to the user ("Resolved via your submission code", "You're already a participant in X", "Only one hackathon is open: X"). |
| null | `0` | No hackathon is currently open for submissions. Tell the user, **stop**, and do not tick `submit`. |
| null | `1` | One hackathon is open but the user isn't bound and didn't supply a code. Ask: `"The only open hackathon is '<name>' (deadline <submission_deadline>). Submit to this one? If so, paste the submission_code the organizer gave you."` Re-run prep with the code. |
| null | `≥ 2` | **Multiple open hackathons — ask the user which one.** Present the list (name, slug, deadline) and ask: `"Which hackathon are you submitting to? Paste its submission_code."` Re-run prep with the code. Never guess. |

> Anti-pattern: do not re-run prep in a loop hoping it resolves — it won't until the user provides a `submission_code`. One question, then re-prep.

### Step 2 — Walk the returned field_schema with the user

`matched.field_schema.fields` is the **authoritative** list of fields for *this* hackathon. Walk every field in order. For each:

1. Show the user the field's `label` (not `key`) and `description`.
2. Propose a value drawn from journey artifacts where obvious:
   - URL-typed fields (`is_url: true` or `type: "url"`) for the deployed app → `deployed_url` from `00-state.md`.
   - Project name / title field → top candidate from `01-idea.md`.
   - Description / summary field → one-liner from `01-idea.md` + must-haves.
   - Repo URL → ask, no default.
   - Team info → ask, no default.
   - Demo video → ask, allow skip if `required: false`.
   - Anything else → ask.
3. Confirm one at a time. Do not batch. Required fields must have a non-empty value before moving on.
4. For `options`-typed fields, present the allowed choices verbatim.

Use `next_call.arguments.data` from the prep response as the literal template — it has one key per field with a placeholder string. Replace each placeholder with the confirmed value.

### Step 3 — Submit

Show the assembled `data` object back to the user verbatim and ask: `"Submit now? (yes/no)"`. On `yes`:

```jsonc
{
  "tool": "prep_and_submit_hackathon_entry",
  "arguments": {
    "action": "submit",
    "hackathon_slug": "<matched.slug from prep — REQUIRED when multiple are open>",
    "app_id": "<app_id from 00-state.md — strongly recommended, unlocks +50 scoring points>",
    "submission_code": "<same code passed to prep, if any — required on first submission>",
    "data": { /* user-confirmed values keyed by field.key */ }
  }
}
```

> Always pass `hackathon_slug = matched.slug` from prep. Without it, submit re-resolves and may target a different hackathon if more than one is open.
> Always pass `app_id` — scoring awards up to 50 points for Butterbase usage on that specific app.

### Step 4 — Receipt

Capture the response (`submission.id`, `submission.version`, `submission.updated_at`, `participant_created`) and write `docs/butterbase/05-submission.md`:

```markdown
# Submission

- submitted_at: <submission.updated_at>
- submission_id: <submission.id>
- version: <submission.version>
- hackathon_slug: <submission.hackathon_slug>
- hackathon_name: <matched.name>
- app_id: <submission.app_id>
- participant_created: <true|false>

## Fields submitted
<one bullet per field: label → value>
```

Tick `- [x] submit` in `00-state.md`, set `current_stage: done`. Print a one-line success to the user including the hackathon name and submission id.

## Outputs

- Hackathon submission via `prep_and_submit_hackathon_entry` (`action: "prep"` then `"submit"`).
- `docs/butterbase/05-submission.md`.

## Anti-patterns

- ❌ Skipping `action: "prep"` and calling `submit` directly with a hardcoded field list.
- ❌ Assuming there is exactly one open hackathon. Always check `open_hackathons` on the prep response.
- ❌ Picking a hackathon for the user when prep returns multiple — always ask.
- ❌ Inventing field keys or labels — only use what's in `matched.field_schema.fields`.
- ❌ Submitting without `hackathon_slug` from prep (can re-resolve to the wrong hackathon).
- ❌ Submitting without `app_id` when one exists (forfeits up to 50 scoring points).
- ❌ Calling submit without explicit `yes` from the user on the assembled payload.
