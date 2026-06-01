---
name: journey-schema
description: Use as the schema build stage of the Butterbase journey, after journey-plan and journey-preflight. Implements the Tables section of 02-plan.md by delegating to schema-design, previewing the diff (manage_schema dry_run) and applying it (manage_schema apply). In hackathon_mode, also folds in the RLS stage by invoking journey-rls inline before returning.
---

# Journey: Schema

Stage 3a of the guided journey. Apply the declarative table definitions from the plan.

## When to use

- Dispatched by `journey` when `current_stage: schema`.
- Directly via `/butterbase-skills:journey-schema`.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the Tables section.
- `docs/butterbase/00-state.md` — for `app_id`, `hackathon_mode`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "schema"`. If the plan calls for advanced schema features (vectors, partial indexes, generated columns), also WebFetch `https://docs.butterbase.ai/schema`. Skip this step if `docs/butterbase/03b-docs-cache.md` was written less than 30 minutes ago and already covers `schema`.

1. Read the Tables section of `02-plan.md`. Print it back: `"About to apply schema: <N tables>. Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:schema-design` via the Skill tool, passing the Tables section and `app_id`. The wrapped skill will build the declarative schema, dry-run it with `manage_schema action: dry_run`, then apply with `action: apply` after user approval of the diff.
3. After it returns, optionally run a sanity `manage_schema action: get` and show the user the live shape.
4. **Hackathon mode only:** if `00-state.md` has `hackathon_mode: true`, also invoke `butterbase-skills:journey-rls` inline now. After it returns, mark the rls checkbox as `(folded)` in `00-state.md` rather than leaving it unchecked.
5. Append one line to `docs/butterbase/04-build-log.md` (create if absent):
   `<ISO timestamp>  schema  manage_schema  ok`
6. Tick `- [x] schema` in `00-state.md`, set `current_stage:` to the next unchecked stage (typically `rls` outside hackathon mode, otherwise the stage after rls), bump `last_updated`.
7. Return to `journey` orchestrator (or, if invoked directly, ask: `"Continue to the next stage? (yes/no)"`).

## Outputs

- Live schema in the Butterbase app.
- One line in `04-build-log.md`.
- Updated `00-state.md`.

## Anti-patterns

- ❌ Calling `manage_schema action: apply` without showing the user the dry-run diff.
- ❌ Forgetting `_drop:` / `_dropColumns:` are needed for destructive changes — never set them without explicit user confirmation.
- ❌ Forgetting the hackathon-mode RLS fold.
