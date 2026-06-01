---
name: journey-substrate
description: Use as the optional substrate-linking stage of the Butterbase journey, after deploy and before submit. Asks whether to connect the deployed app to the owner's substrate (so functions get ctx.substrate). Skipped by default in hackathon mode.
---

# Journey Substrate Stage (optional)

Connect the deployed app to the owner's substrate so functions get `ctx.substrate` injected at cold start. Skip if the user has no AI-agent / memory use case.

## When to use

Invoke automatically when the journey orchestrator's cursor reaches `substrate`. The row is optional in the checklist; skip silently if the user declines.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "substrate"`. If the plan mentions AI memory / agent state / cross-session knowledge, also WebFetch `https://docs.butterbase.ai/substrate`. Skip if `docs/butterbase/03b-docs-cache.md` already covers `substrate`.

1. **Confirm.** Ask the user:
   > "Connect `<app_id>` to your substrate? This lets the app's functions read/write your agent memory via `ctx.substrate`. You can disable later. (yes / skip)"

   Default: **skip**.

2. **On skip:** mark the row `- [x] substrate (skipped — no agent memory needed)` in `docs/butterbase/00-state.md`. Write `docs/butterbase/04b-substrate.md` with one line: `Skipped on <date>.` Return.

3. **On yes:**
   a. If `platform_users.substrate_provisioned_at` is NULL for the caller, call `POST /v1/me/substrate/provision` first (or invoke whatever MCP wrapper exists — check `butterbase_docs` topic `substrate`).
   b. Link the app: `POST /v1/apps/<app_id>/substrate` with body `{"substrate_user_id": "<owner_id>"}`. (The control-API enforces `substrate_user_id == owner_id` in v1.)
   c. Verify: `GET /v1/apps/<app_id>` and assert `substrate_user_id` matches.
   d. Smoke: invoke any HTTP function the app already has and check `ctx.substrate` is defined (if the function logs it). If no function exists yet, skip the smoke and note in the artifact.

4. **Write artifact.** `docs/butterbase/04b-substrate.md`:

   ```markdown
   ---
   linked_at: <ISO>
   app_id: <id>
   substrate_user_id: <id>
   ---

   # Substrate Linkage

   - App linked at <ISO>
   - Substrate user ID: <id>
   - Smoke: <pass / skipped — no function yet>
   ```

5. **Update state.** Tick the `substrate` row in `00-state.md`.

## Anti-patterns

- ❌ Auto-linking without asking. Substrate is a privacy-sensitive surface.
- ❌ Linking before the app has any function deployed. Wait until `journey-functions` has run.
- ❌ Forgetting to handle the "substrate not yet provisioned" case. Lazy provisioning is by design.
