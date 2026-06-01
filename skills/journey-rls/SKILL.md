---
name: journey-rls
description: Use as the RLS build stage of the Butterbase journey, after journey-schema. Implements the RLS section of 02-plan.md by delegating to debug-rls policy patterns (for proactive creation, not debugging). Calls manage_rls (create_user_isolation, enable, create_policy). Folded into journey-schema when hackathon_mode is true.
---

# Journey: RLS

Stage 3b of the guided journey. Install Row-Level Security policies for user-owned tables.

## When to use

- Dispatched by `journey` when `current_stage: rls`.
- Directly via `/butterbase-skills:journey-rls`.
- Folded into `journey-schema` when `hackathon_mode: true` (do not run separately).

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the RLS section.
- `docs/butterbase/00-state.md` — for `app_id`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "auth"`. For RLS-specific patterns, also WebFetch `https://docs.butterbase.ai/auth/rls`. Skip if cache is fresh.

1. Read the RLS section of `02-plan.md`. Print it back: `"About to install RLS policies: <list>. Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:debug-rls` via the Skill tool with mode `proactive`, passing the RLS plan and `app_id`. For each user-isolation entry, the wrapped skill calls `manage_rls action: create_user_isolation`. For custom policies, `manage_rls action: enable` then `action: create_policy`.
3. After it returns, sanity-check with `manage_rls action: list` and show the user.
4. Append one line to `docs/butterbase/04-build-log.md`:
   `<ISO timestamp>  rls  manage_rls  ok`
5. Tick `- [x] rls` in `00-state.md`, set `current_stage:` to the next unchecked stage, bump `last_updated`.
6. Return to `journey` orchestrator (or ask `"Continue to the next stage? (yes/no)"`).

## Outputs

- Live RLS policies in the Butterbase app.
- One line in `04-build-log.md`.

## Anti-patterns

- ❌ Skipping `manage_rls action: list` verification — invisible policy failures are the #1 RLS gotcha.
- ❌ Creating policies on a table where RLS is not enabled — `enable` must come first.
