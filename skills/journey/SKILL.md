---
name: journey
description: Use when the user says "build an app", "let's start", "help me build", "I have an idea for", "ship it", or otherwise signals they want to go from idea to deployed Butterbase app. Orchestrates the full guided journey (idea → plan → preflight → build → deploy → optional hackathon submit) by reading docs/butterbase/00-state.md and dispatching to the next stage skill.
---

# Butterbase Guided Journey

End-to-end orchestrator. Walks the user from idea to deployed Butterbase app, and (in hackathon mode) on through submission. Each stage writes a markdown artifact under `docs/butterbase/` in the user's project. The orchestrator reads `docs/butterbase/00-state.md` to know the cursor and dispatches the matching `journey-*` stage skill.

## When to use

Invoke automatically when the user signals end-to-end intent: "I want to build…", "let's build an app", "ship this", "help me build a hackathon project". Invoke explicitly when the user runs `/butterbase-skills:journey`.

If the user wants to do a single stage only (e.g., just design a schema), defer to the matching standalone skill (`schema-design`) or per-stage command (`/butterbase-skills:journey-schema`) instead of starting the full journey.

## Toolchain

The journey assumes two npm packages alongside the MCP tools:

| Package | Role | When to install |
|---|---|---|
| `@butterbase/sdk` | TypeScript client for the deployed app — `auth.signIn`, `db.from(...).select()`, `storage.upload`, realtime subscriptions, function invocation. Works both in the browser (frontend) and in Node (functions, scripts, server-side). | Install in any frontend or any Node service that talks to the deployed app. Auto-added by frontend scaffolds. |
| `@butterbase/cli` | Local-dev CLI — project scaffolding, log tailing, function invocation from the shell, API-key generation, schema diff preview without the dashboard. | Install globally (`npm i -g @butterbase/cli`) once per dev machine. Used during `journey-preflight`. |

The journey will prompt for both in `plan` (which SDK surfaces does the app need?) and `preflight` (is the CLI installed?). The MCP tools and these packages are complementary: MCP is for agentic / orchestration flows; SDK + CLI are for ordinary application code and the human dev loop.

## Procedure

1. **Detect state.** Check whether `docs/butterbase/00-state.md` exists in the working directory.
   - If absent: this is a fresh journey. Ask the user `"Is this a hackathon submission? (yes/no)"` to set `hackathon_mode`. Create `docs/butterbase/` and write a starter `00-state.md` (template below). If `hackathon_mode: false`, the starter template's `submit` row must be written as `- [ ] submit (n/a — not a hackathon)` rather than plain `- [ ] submit`. Then proceed to stage `idea`.
   - If present: read the front-matter and the stage checklist. Identify the first unchecked, non-skipped stage. That is the next stage.

2. **Confirm with user.** Print a one-line summary of where we are: `"Resuming journey at <stage> for app_id <id or 'not yet provisioned'>."` Ask: `"Continue from <stage>? (yes / jump to other stage / redo previous)"`.

3. **Dispatch.** Invoke the matching skill via the Skill tool. Do not do the stage's work inline — delegate.

   | Stage | Skill |
   |---|---|
   | idea | `butterbase-skills:journey-idea` |
   | plan | `butterbase-skills:journey-plan` |
   | preflight | `butterbase-skills:journey-preflight` |
   | docs | `butterbase-skills:journey-docs` |
   | schema | `butterbase-skills:journey-schema` |
   | rls | `butterbase-skills:journey-rls` |
   | auth | `butterbase-skills:journey-auth` |
   | storage | `butterbase-skills:journey-storage` |
   | functions | `butterbase-skills:journey-functions` |
   | ai | `butterbase-skills:journey-ai` |
   | rag | `butterbase-skills:journey-rag` |
   | realtime | `butterbase-skills:journey-realtime` |
   | durable | `butterbase-skills:journey-durable` |
   | agents | `butterbase-skills:journey-agents` |
   | frontend | `butterbase-skills:journey-frontend` |
   | deploy | `butterbase-skills:journey-deploy` |
   | substrate | `butterbase-skills:journey-substrate` (optional) |
   | submit | `butterbase-skills:journey-submit` (hackathon_mode only) |
   | templates | `butterbase-skills:journey-templates` (optional) |

**Docs gate.** Stage `docs` runs once, right after preflight, to prime `butterbase_docs` for every capability in the plan. Subsequent build stages start with the relevant docs cached at `docs/butterbase/03b-docs-cache.md`. If the user changes the plan mid-build, re-run `/butterbase-skills:journey-docs` before the affected stage.

4. **After the stage skill returns,** re-read `00-state.md` and ask the user whether to advance to the next unchecked stage. Stage selection rules:
   - Build-stage order is: schema → rls → auth → storage → functions → ai → rag → realtime → durable → **agents** → frontend.
   - If `hackathon_mode: true` and all build stages are done, the next stage is `deploy` then `substrate` then `submit` then `templates`.
   - If `hackathon_mode: false`, the journey ends at `deploy` unless the user opts into `templates` — skip `substrate` and `submit` entirely (treat them as `(n/a)`).
   - The `substrate` stage is always optional. In hackathon mode, default to skipping it (mark as `(n/a — optional, can add post-submission)`); in non-hackathon mode, skip it entirely as it is not part of the core journey.
   - The `templates` stage is always optional. It runs after `deploy` (non-hackathon) or after `submit` (hackathon). Default is to skip unless `01-idea.md` or `02-plan.md` flags `publish_as_template: true`.
   - Loop until the cursor reaches `DONE` (every stage checked or annotated `n/a`).

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
- [ ] docs
- [ ] schema
- [ ] rls
- [ ] auth
- [ ] storage
- [ ] functions
- [ ] ai
- [ ] rag
- [ ] realtime
- [ ] durable
- [ ] agents
- [ ] frontend
- [ ] deploy
- [ ] substrate (optional)
- [ ] submit
- [ ] templates (optional)

## Notes
- Journey initialised <ISO date>.
```

If `hackathon_mode: false`, write the submit row as `- [ ] submit (n/a — not a hackathon)` instead of the plain unchecked form, so the orchestrator and any direct stage invocation will skip it.

## Outputs

- Creates `docs/butterbase/` and `00-state.md` on first run.
- Updates `current_stage` and `last_updated` in `00-state.md` whenever it dispatches.

## Anti-patterns

- ❌ Doing a stage's work inline instead of delegating to its skill. The orchestrator only routes.
- ❌ Skipping the "Continue from <stage>?" confirmation — users may want to jump or redo.
- ❌ Letting `00-state.md` drift out of sync. Always re-read after each stage returns.
