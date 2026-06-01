---
name: journey-plan
description: Use as stage 2 of the Butterbase journey, after journey-idea has written 01-idea.md. Translates the idea + capability map into a concrete Butterbase plan — tables (with columns/types/RLS shape), auth providers, function list (name + trigger), storage buckets, AI/RAG/realtime/durable usage, and the chosen frontend stack. In hackathon mode, ruthlessly cuts scope into a "ship now" vs "post-hackathon" split. Produces docs/butterbase/02-plan.md.
---

# Journey: Plan

Stage 2 of the guided journey. Turn the idea brief into an actionable Butterbase plan.

## When to use

- Dispatched by `journey` when `current_stage: plan`.
- Directly via `/butterbase-skills:plan`.

## Inputs

- `docs/butterbase/01-idea.md` (must exist — if absent, bounce back to `journey-idea`).
- `docs/butterbase/00-state.md` (for `hackathon_mode`, `hackathon_deadline`).

### External services — check built-ins first

When the plan needs email, messaging, calendar, CRM, docs, or project-management integration:

1. Invoke `butterbase-skills:integrations` and call `manage_integrations` `action: "list_available"` to see what Composio covers for this app.
2. If a toolkit fits, the plan should record "via manage_integrations (toolkit: <name>)" instead of naming an external SDK.

When the plan needs payments:

1. Invoke `butterbase-skills:payments`.
2. Default to Stripe Connect via `manage_billing` unless the user's region forces a fallback (see the payments skill).
3. Record the choice ("Stripe Connect" or "<regional gateway> via function proxy") in the plan's Payments section.

## Procedure

Work through these sections in order. After each section, write the result to `02-plan.md` before moving on. One question at a time per the spec's questioning discipline.

1. **Tables.** Read the capability map. Propose a starter table list with columns and types — recommend, don't ask blank. Example: `"Tables I'm seeing: users, orders, items. Missing any?"` Then for each table: `"<table>.<column>: should this be a uuid / text / int / timestamp / enum?"`. Confirm primary keys, foreign keys, indexes that are obvious (foreign-key columns).

2. **RLS model.** For each table: `"Can user A see user B's <table> rows? ① no, strict isolation ② yes, public-read ③ only shared via explicit grant."` Decide policy shape. In hackathon mode, prefer option ① and recommend `manage_rls action: create_user_isolation`.

3. **Auth.** `"OAuth providers: ① Google only ② Google + GitHub ③ email/password too ④ none (anonymous app)."` Also ask: `"Need a demo / judge account seeded? (hackathon mode only)"`.

4. **Functions.** For each function from the capability map: `"<name>: trigger = HTTP / cron / WebSocket? If cron: schedule? If HTTP: idempotency needed?"`.

5. **Storage.** If used: `"Which objects (avatars, attachments, …)? Public-read or per-user?"`.

6. **AI / RAG / realtime / durable.** Only if used. Capture model choice (AI), collections (RAG), tables to subscribe to (realtime), object kinds (durable).

7. **Frontend stack.** `"Frontend: ① Vite + React ② Next.js ③ static HTML ④ none (API-only)."` Write to `00-state.md` `frontend_stack`.

### Toolchain question

Ask: "Will your app's code use `@butterbase/sdk` only in the frontend, only server-side (functions, scripts), or both?" Record under Toolchain → SDK surfaces.

Ask: "Want to install `@butterbase/cli` for the local dev loop (logs, scaffolding, key rotation)? (yes/no — default yes)" Record under Toolchain → CLI usage.

8. **Scope cut (hackathon mode).** Re-read the must-haves list. For each, ask: `"Ship now or post-hackathon?"` Write the cut list into `02-plan.md`'s "Post-hackathon" section.

9. **Annotate skipped stages.** For every build stage NOT used in this plan (check the capability map and feature list), update `00-state.md`'s checklist to read `- [ ] <stage> (n/a)` for that row. Also do this for `rls` if `hackathon_mode: true` (mark as `(folded into schema)`).

10. **Final approval.** Show the user the assembled plan and ask: `"Plan looks good? (yes / revise <section>)"`. Loop until yes.

## `02-plan.md` format

```markdown
# Plan

## Tables
- `users` (id uuid pk, email text unique, created_at timestamp)
- `orders` (id uuid pk, user_id uuid fk→users.id, status enum[pending,paid,shipped], total int, created_at timestamp; index on user_id)
- ...

## RLS
- `orders`: user-isolation (create_user_isolation, owner column = user_id)
- ...

## Auth
- Providers: Google
- Demo user: yes (email demo@example.com, password set via seed)

## Functions
- `stripe-webhook` — HTTP, idempotency table `_processed_events`
- `daily-digest` — cron 0 9 * * *  UTC

## Storage
- bucket: `avatars` (per-user, private; download via presigned URL)

## AI / RAG / realtime / durable
- (none)

## Frontend
- Vite + React

## Toolchain

- **SDK surfaces**: <client-side only | server-side only | both>
  - Client-side: install `@butterbase/sdk` in the frontend; use `auth`, `db`, `storage`, `realtime`.
  - Server-side: install `@butterbase/sdk` in functions / scripts; use the service-key flow for elevated access.
- **CLI usage**: <yes / no>
  - Yes (default): use `butterbase` CLI for local scaffolding, log tailing (`butterbase logs <fn>`), function invocation, and key rotation.
- **Why both**: MCP tools provision and orchestrate; SDK + CLI are the runtime + dev loop.

## Build order
1. schema
2. rls (folded into schema in hackathon mode)
3. auth
4. storage
5. functions
6. ai          (if used)
7. rag         (if used)
8. realtime    (if used)
9. durable     (if used)
10. frontend
11. deploy

## Post-hackathon
- email notifications (deferred)
- admin dashboard (deferred)
```

## Outputs

- Writes `docs/butterbase/02-plan.md`.
- Updates `00-state.md`: tick `- [x] plan`, set `frontend_stack`, set `current_stage: preflight`, annotate skipped build stages with `(n/a)`.

## Anti-patterns

- ❌ Asking the user to design every table column without recommending defaults first.
- ❌ Skipping the scope-cut section in hackathon mode.
- ❌ Writing the plan only at the end — write as you go so progress survives a crash.
