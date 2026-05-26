---
name: journey
description: Use when the user says "build an app", "let's start", "help me build", "I have an idea for", "ship it", or otherwise signals they want to go from idea to deployed Butterbase app. Orchestrates the full guided journey (idea → plan → preflight → build → deploy → optional hackathon submit) by reading docs/butterbase/00-state.md and dispatching to the next stage skill.
---

# Butterbase Guided Journey

End-to-end orchestrator. Walks the user from idea to deployed Butterbase app, and (in hackathon mode) on through submission. Each stage writes a markdown artifact under `docs/butterbase/` in the user's project. The orchestrator reads `docs/butterbase/00-state.md` to know the cursor and dispatches the matching `journey-*` stage skill.

## When to use

Invoke automatically when the user signals end-to-end intent: "I want to build…", "let's build an app", "ship this", "help me build a hackathon project". Invoke explicitly when the user runs `/butterbase:journey`.

If the user wants to do a single stage only (e.g., just design a schema), defer to the matching standalone skill (`schema-design`) or per-stage command (`/butterbase:journey-schema`) instead of starting the full journey.

## Procedure

1. **Detect state.** Check whether `docs/butterbase/00-state.md` exists in the working directory.
   - If absent: this is a fresh journey. Create `docs/butterbase/` and write a starter `00-state.md` (template below). Then proceed to stage `idea`.
   - If present: read the front-matter and the stage checklist. Identify the first unchecked, non-skipped stage. That is the next stage.

2. **Confirm with user.** Print a one-line summary of where we are: `"Resuming journey at <stage> for app_id <id or 'not yet provisioned'>."` Ask: `"Continue from <stage>? (yes / jump to other stage / redo previous)"`.

3. **Dispatch.** Invoke the matching skill via the Skill tool. Do not do the stage's work inline — delegate.

   | Stage | Skill |
   |---|---|
   | idea | `butterbase:journey-idea` |
   | plan | `butterbase:journey-plan` |
   | preflight | `butterbase:journey-preflight` |
   | schema | `butterbase:journey-schema` |
   | rls | `butterbase:journey-rls` |
   | auth | `butterbase:journey-auth` |
   | storage | `butterbase:journey-storage` |
   | functions | `butterbase:journey-functions` |
   | ai | `butterbase:journey-ai` |
   | rag | `butterbase:journey-rag` |
   | realtime | `butterbase:journey-realtime` |
   | durable | `butterbase:journey-durable` |
   | frontend | `butterbase:journey-frontend` |
   | deploy | `butterbase:journey-deploy` |
   | submit | `butterbase:journey-submit` |

4. **After the stage skill returns,** re-read `00-state.md` and ask the user whether to advance to the next unchecked stage. If `hackathon_mode: true` and all build stages are done, the next stage is `deploy` then `submit`. Loop until the cursor reaches `DONE` (every stage checked or annotated `n/a`).

## Starter `00-state.md` template

When initialising a fresh journey, write this to `docs/butterbase/00-state.md` (ask the user `"Is this a hackathon submission? (yes/no)"` first to set `hackathon_mode`):

```markdown
---
app_id: null
api_base: null
hackathon_mode: <true|false>
hackathon_deadline: null
frontend_stack: null
current_stage: idea
last_updated: <ISO-8601 timestamp>
---

# Journey state

## Stages
- [ ] idea
- [ ] plan
- [ ] preflight
- [ ] schema
- [ ] rls
- [ ] auth
- [ ] storage
- [ ] functions
- [ ] ai
- [ ] rag
- [ ] realtime
- [ ] durable
- [ ] frontend
- [ ] deploy
- [ ] submit

## Notes
- Journey initialised <ISO date>.
```

## Outputs

- Creates `docs/butterbase/` and `00-state.md` on first run.
- Updates `current_stage` and `last_updated` in `00-state.md` whenever it dispatches.

## Anti-patterns

- ❌ Doing a stage's work inline instead of delegating to its skill. The orchestrator only routes.
- ❌ Skipping the "Continue from <stage>?" confirmation — users may want to jump or redo.
- ❌ Letting `00-state.md` drift out of sync. Always re-read after each stage returns.
