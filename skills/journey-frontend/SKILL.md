---
name: journey-frontend
description: Use as the frontend build stage of the Butterbase journey. Implements the Frontend section of 02-plan.md by delegating to deploy-frontend. Scaffolds (if needed) and deploys via create_frontend_deployment + manage_frontend (start_deployment). Sets VITE_API_URL and VITE_APP_ID env. Skipped if the plan is API-only.
---

# Journey: Frontend

Stage 3j of the guided journey. Build (or adopt) and deploy the frontend.

## When to use

- Dispatched by `journey` when `current_stage: frontend`.
- Directly via `/butterbase-skills:journey-frontend`.
- Skipped (annotated `(n/a)`) if `frontend_stack: none` in `00-state.md`.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the Frontend section.
- `docs/butterbase/00-state.md` — for `app_id`, `api_base`, `frontend_stack`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "frontend"`. For framework-specific deploy patterns, also WebFetch `https://docs.butterbase.ai/frontend`. Skip if cache is fresh.

### Use `@butterbase/sdk`

Any frontend that talks to the deployed Butterbase app should use `@butterbase/sdk`:

- Install: `npm install @butterbase/sdk` inside the frontend project.
- Initialize: `import { createClient } from '@butterbase/sdk'; const bb = createClient({ apiUrl: import.meta.env.VITE_API_URL, appId: import.meta.env.VITE_APP_ID });`
- Auth: `bb.auth.signInWithOAuth({ provider: 'google' })`, `bb.auth.signOut()`, `bb.auth.getSession()`.
- Data: `bb.db.from('posts').select('*').eq('user_id', userId)`.
- Storage: `bb.storage.upload(file)`, persist the returned `object_id`, resolve download URLs via the SDK at render time.
- Realtime: `bb.realtime.from('posts').on('insert', cb).subscribe()`.

Do NOT hand-roll `fetch()` against the REST API in a Butterbase frontend — the SDK handles auth headers, presigned URL refresh, realtime reconnection, and type-narrowing. The MCP tools are not for runtime code.

For framework-specific patterns (Next.js Server Components, SvelteKit load functions, etc.), call `butterbase_docs` with `topic: "sdk"`.

### Build and deploy

1. Read the Frontend section and `frontend_stack` from `00-state.md`. Print: `"About to deploy a <stack> frontend for app_id <id>. Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:deploy-frontend` via the Skill tool with the frontend spec, `app_id`, and `api_base`. The wrapped skill scaffolds (if no `package.json` exists in `./web` or chosen path), sets `VITE_API_URL` and `VITE_APP_ID`, builds, calls `create_frontend_deployment`, then `manage_frontend action: start_deployment`.
3. Capture the live URL from the response. Show it to the user.
4. Append one line to `docs/butterbase/04-build-log.md`:
   `<ISO timestamp>  frontend  manage_frontend  <live-url>  ok`
5. Also write the live URL into a new `deployed_url:` field in `00-state.md` front-matter.
6. Tick `- [x] frontend` in `00-state.md`, set `current_stage: deploy`.
7. Return to `journey` orchestrator (or ask `"Continue to deploy verification? (yes/no)"`).

## Outputs

- Live deployed frontend URL.
- `deployed_url:` in `00-state.md`.
- One line in `04-build-log.md`.

## Anti-patterns

- ❌ Forgetting to set `VITE_API_URL` and `VITE_APP_ID` — frontend will load blank.
- ❌ Wrong MIME types — let `deploy-frontend` handle the bundling; do not hand-roll.
