---
name: journey-realtime
description: Use as the realtime build stage of the Butterbase journey. Implements the Realtime section of 02-plan.md by delegating to the realtime skill. Calls manage_realtime (configure) to enable WebSocket subscriptions on the listed tables. Skipped if the plan has no realtime needs.
---

# Journey: Realtime

Stage 3h of the guided journey. Enable WebSocket subscriptions per the plan.

## When to use

- Dispatched by `journey` when `current_stage: realtime`.
- Directly via `/butterbase-skills:journey-realtime`.
- Skipped (annotated `(n/a)`) if the plan has no Realtime section.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the Realtime section.
- `docs/butterbase/00-state.md` — for `app_id`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "realtime"`. For WebSocket subscription shape and RLS interaction, also WebFetch `https://docs.butterbase.ai/realtime`. Skip if cache is fresh.

1. Read the Realtime section. Print it back: `"About to enable realtime on: <tables>. Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:realtime` via the Skill tool with the realtime plan and `app_id`. The wrapped skill calls `manage_realtime action: configure` per table and reminds the user that RLS still applies to subscribed rows.
3. Verify with `manage_realtime action: get`.
4. Append one line to `docs/butterbase/04-build-log.md`:
   `<ISO timestamp>  realtime  manage_realtime  ok`
5. Tick `- [x] realtime` in `00-state.md`, set `current_stage:` to the next unchecked stage.
6. Return to `journey` orchestrator (or ask `"Continue to the next stage? (yes/no)"`).

## Outputs

- Realtime configured tables.
- One line in `04-build-log.md`.

## Anti-patterns

- ❌ Enabling realtime on a table whose RLS hasn't been configured — clients will connect but see no events.
