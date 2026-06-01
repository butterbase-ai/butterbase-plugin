---
name: journey-auth
description: Use as the auth build stage of the Butterbase journey, after journey-schema (and rls if separate). Implements the Auth section of 02-plan.md by delegating to auth-setup. Calls manage_oauth (configure) for providers and optionally manage_auth_config (configure_auth_hook, update_jwt). Skipped if the plan has no end-user auth.
---

# Journey: Auth

Stage 3c of the guided journey. Configure OAuth providers and (optionally) auth hooks.

## When to use

- Dispatched by `journey` when `current_stage: auth`.
- Directly via `/butterbase-skills:journey-auth`.
- Skipped (annotated `(n/a)` in `00-state.md`) if the plan has no end-user auth.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the Auth section.
- `docs/butterbase/00-state.md` — for `app_id`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "auth"`. For provider-specific setup (Google/GitHub/Apple), also WebFetch `https://docs.butterbase.ai/auth`. Skip if cache is fresh.

1. Read the Auth section of `02-plan.md`. Print it back: `"About to configure auth: providers=<list>, demo_user=<yes/no>. Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:auth-setup` via the Skill tool, passing the Auth plan and `app_id`. The wrapped skill will prompt for each provider's client ID/secret and call `manage_oauth action: configure`. If a demo user is requested, it seeds one via `insert_row` on the users table.
3. After it returns, run `manage_oauth action: get` to confirm and print the redirect URLs the user must register with each provider.
4. Append one line to `docs/butterbase/04-build-log.md`:
   `<ISO timestamp>  auth  manage_oauth  ok`
5. Tick `- [x] auth` in `00-state.md`, set `current_stage:` to the next unchecked stage.
6. Return to `journey` orchestrator (or ask `"Continue to the next stage? (yes/no)"`).

## Outputs

- Configured OAuth providers.
- Optional seed user.
- One line in `04-build-log.md`.

## Anti-patterns

- ❌ Echoing OAuth client secrets back to the user.
- ❌ Forgetting to give the user the provider-side redirect URL — auth will silently fail without it.
