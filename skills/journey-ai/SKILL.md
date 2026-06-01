---
name: journey-ai
description: Use as the AI build stage of the Butterbase journey. Implements the AI section of 02-plan.md by delegating to the ai skill. Calls manage_ai (update_config) to set defaults and optionally BYOK. Skipped if the plan has no LLM/embeddings usage.
---

# Journey: AI

Stage 3f of the guided journey. Configure AI gateway defaults (model, BYOK).

## When to use

- Dispatched by `journey` when `current_stage: ai`.
- Directly via `/butterbase-skills:journey-ai`.
- Skipped (annotated `(n/a)`) if the plan has no AI section.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the AI section.
- `docs/butterbase/00-state.md` — for `app_id`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "ai"`. For BYOK and model lists, also WebFetch `https://docs.butterbase.ai/ai`. Skip if cache is fresh.

1. Read the AI section. Print it back: `"About to configure AI: default model=<m>, BYOK=<yes/no>. Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:ai` via the Skill tool with the AI plan and `app_id`. The wrapped skill calls `manage_ai action: update_config` (and per-provider key updates if BYOK).
3. Smoke: call `manage_ai action: chat` with a tiny prompt (`"say ok"`) to confirm the gateway responds.
4. Append one line to `docs/butterbase/04-build-log.md`:
   `<ISO timestamp>  ai  manage_ai  ok`
5. Tick `- [x] ai` in `00-state.md`, set `current_stage:` to the next unchecked stage.
6. Return to `journey` orchestrator (or ask `"Continue to the next stage? (yes/no)"`).

## Outputs

- Configured AI gateway defaults.
- One line in `04-build-log.md`.

## Anti-patterns

- ❌ Echoing BYOK keys back to the user.
- ❌ Picking a model not in `manage_ai action: list_models` — list first.
