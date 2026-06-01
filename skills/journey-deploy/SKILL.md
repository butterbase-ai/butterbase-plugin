---
name: journey-deploy
description: Use as the deploy-verification stage of the Butterbase journey, after journey-frontend (or after any build stage if there is no frontend). Smoke-tests the deployed app — hits the live URL, invokes any deployed functions, checks auth round-trip. Writes results to docs/butterbase/04-build-log.md. Blocks journey-submit if smoke fails.
---

# Journey: Deploy verification

Stage 4 of the guided journey. Verify the deployed app actually works end-to-end.

## When to use

- Dispatched by `journey` when `current_stage: deploy`.
- Directly via `/butterbase-skills:journey-deploy`.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/00-state.md` — for `app_id`, `deployed_url`.
- `docs/butterbase/02-plan.md` — for the function list to smoke.
- `docs/butterbase/04-build-log.md` — to know what shipped.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "frontend"`. For end-to-end smoke patterns, also WebFetch `https://docs.butterbase.ai/deploy`. Skip if cache is fresh.

For each check, log a one-line result to `04-build-log.md`. Halt on the first ✗ and ask the user to fix or skip.

1. **Frontend reachable.** If `deployed_url` is set, fetch it via Bash `curl -sS -o /dev/null -w "%{http_code}" <url>`. Expect `200`. Log `<ISO>  deploy  curl  200  ok` or `... <code>  fail`.

2. **Functions respond.** For each function in the plan with HTTP trigger, call `mcp__butterbase__invoke_function` with a representative payload. Expect 2xx. Log per function.

3. **Auth round-trip (if configured).** If the plan included OAuth, prompt the user: `"Open <deployed_url>, click 'Sign in with <provider>', and confirm you land logged-in. Did it work? (yes/no)"`. Log the answer.

4. **Function logs clean.** Call `mcp__butterbase__manage_function action: get_logs` for the most recent invocations. Show the user any error-level lines. Ask: `"Anything concerning here? (no → continue, yes → fix and re-run)"`.

5. If all checks pass, tick `- [x] deploy` in `00-state.md`, set `current_stage: submit` (if `hackathon_mode: true`) or `current_stage: done` (otherwise).

6. Return to `journey` orchestrator.

## Outputs

- Multiple lines in `04-build-log.md`.
- Updated `00-state.md`.

## Anti-patterns

- ❌ Treating a 200 from the frontend root as proof the app works — actually invoke a function.
- ❌ Skipping the auth round-trip check just because OAuth is configured server-side — judges will try to log in.
