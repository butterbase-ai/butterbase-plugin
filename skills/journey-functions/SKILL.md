---
name: journey-functions
description: Use as the functions build stage of the Butterbase journey. Implements the Functions section of 02-plan.md by delegating to function-dev for each function. Calls deploy_function per function; smokes each with invoke_function. Skipped if the plan has no functions.
---

# Journey: Functions

Stage 3e of the guided journey. Implement and deploy each function in the plan.

## When to use

- Dispatched by `journey` when `current_stage: functions`.
- Directly via `/butterbase-skills:journey-functions`.
- Skipped (annotated `(n/a)`) if the plan lists no functions.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the Functions section.
- `docs/butterbase/00-state.md` — for `app_id`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "functions"`. For trigger types and ctx shape, also WebFetch `https://docs.butterbase.ai/functions`. Skip if cache is fresh.

0.5. **Check built-in integrations first.** Before writing function code that calls an external SaaS for email / messaging / calendar / CRM / payments:
   - Email / Slack / etc. → invoke `butterbase-skills:integrations`. The function should call `manage_integrations` `execute_action` rather than installing a third-party SDK.
   - Payments → invoke `butterbase-skills:payments`. The function should use Stripe Connect via `manage_billing` unless the plan has explicitly chosen a regional gateway.

### `@butterbase/sdk` works server-side too

Inside a function, prefer `ctx.db` / `ctx.storage` / `ctx.user` for the common cases — those are pre-wired and authenticated against the calling user. But for cross-app calls, scripts, or scheduled jobs that operate on multiple apps, instantiate `@butterbase/sdk` with a service key (`bb_sk_`) and use the same client surface as the frontend.

Example for a cron function that aggregates from another app:
```ts
import { createClient } from '@butterbase/sdk';

export async function handler(_request: Request, ctx: { env: Record<string,string> }) {
  const other = createClient({
    apiUrl: ctx.env.OTHER_APP_API_URL,
    apiKey: ctx.env.OTHER_APP_SERVICE_KEY,
  });
  const { data } = await other.db.from('events').select('*').gte('created_at', ...);
  // ...
  return new Response('ok');
}
```

For server-side patterns, `butterbase_docs` `topic: "sdk"`.

### Build each function

For each function in the plan, in order:

1. Print: `"About to build function: <name> (trigger=<trigger>). Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:function-dev` via the Skill tool with the function spec (name, trigger, behaviour, dependencies) and `app_id`. The wrapped skill scaffolds the handler, writes tests where appropriate, and calls `deploy_function`. Reminder it must enforce: handler signature `(request, { db, env, user })` and must return `new Response(...)`.
3. Smoke: call `invoke_function` for HTTP/cron functions and confirm a 2xx + expected body. For WebSocket, defer the smoke to frontend integration.
4. Append one line per function to `docs/butterbase/04-build-log.md`:
   `<ISO timestamp>  functions  deploy_function  <fn-name>  ok`
5. After all functions are done, tick `- [x] functions` in `00-state.md`, set `current_stage:` to the next unchecked stage.
6. Return to `journey` orchestrator (or ask `"Continue to the next stage? (yes/no)"`).

## Outputs

- Deployed functions in the Butterbase app.
- One line per function in `04-build-log.md`.

## Anti-patterns

- ❌ Letting a handler return a plain object — must be `new Response(...)`.
- ❌ Skipping the smoke invocation. Cron functions in particular are easy to deploy and forget.
- ❌ Forgetting per-function env vars — use `manage_function action: update_env`.
