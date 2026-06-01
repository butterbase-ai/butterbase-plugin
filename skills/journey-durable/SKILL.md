---
name: journey-durable
description: Use as the durable-objects build stage of the Butterbase journey. Implements the Durable section of 02-plan.md by delegating to durable-objects. Calls manage_durable_objects (deploy). Skipped if the plan has no per-key stateful actors (chat rooms, multiplayer, rate limiters).
---

# Journey: Durable Objects

Stage 3i of the guided journey. Deploy stateful per-key actors.

## When to use

- Dispatched by `journey` when `current_stage: durable`.
- Directly via `/butterbase-skills:journey-durable`.
- Skipped (annotated `(n/a)`) if the plan has no Durable section.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the Durable section.
- `docs/butterbase/00-state.md` — for `app_id`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "functions"`. For Durable Object class structure, also WebFetch `https://docs.butterbase.ai/durable-objects`. Skip if cache is fresh.

For each Durable Object class in the plan:

1. Print: `"About to deploy Durable Object: <name>. Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:durable-objects` via the Skill tool with the DO spec and `app_id`. The wrapped skill calls `manage_durable_objects action: deploy`, sets env via `set_env` if needed.
3. Verify with `manage_durable_objects action: list` and `action: get`.
4. Append one line per DO to `docs/butterbase/04-build-log.md`:
   `<ISO timestamp>  durable  manage_durable_objects  <do-name>  ok`
5. After all DOs are done, tick `- [x] durable` in `00-state.md`, set `current_stage:` to the next unchecked stage.
6. Return to `journey` orchestrator (or ask `"Continue to the next stage? (yes/no)"`).

## Outputs

- Deployed Durable Object classes.
- One line per DO in `04-build-log.md`.

## Anti-patterns

- ❌ Sharing state across users via a single DO key — keys should be naturally per-room / per-user / per-resource.
