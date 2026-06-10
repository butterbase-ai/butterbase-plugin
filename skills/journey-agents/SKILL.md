---
name: journey-agents
description: Use as the agents build stage of the Butterbase journey. Implements the Agents section of 02-plan.md by delegating to the `agents` skill for each agent. Registers any required MCP servers, validates each graph_spec, creates the agent, and smokes it via invoke_agent. Skipped if the plan has no agents.
---

# Journey: Agents

Build the agents described in `02-plan.md`. Order: MCP servers first (probe must pass before any agent references them) → agents → smoke runs.

## When to use

- Dispatched by `journey` when `current_stage: agents`.
- Directly via `/butterbase-skills:journey-agents`.
- Skipped (annotated `(n/a)`) if the plan lists no agents.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the Agents section (names, purpose, tool list, visibility, MCP servers needed).
- `docs/butterbase/00-state.md` — for `app_id`.
- Any handcrafted spec files under `agents/` in the project (preferred — version-controlled and survives clones).

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "agents"`. If the cache (`docs/butterbase/03b-docs-cache.md`) already covers agents, skip.

1. **Register MCP servers first.** For each MCP server in the plan:
   - Print: `"Registering MCP server '<label>' at <url> (transport=<sse|http|streamable_http>). Proceed?"`. Wait for `yes`.
   - Call the MCP-servers registration route (or the dashboard's MCP Servers page if a wrapping MCP tool is not yet available — check `butterbase_docs` topic `agents` for the current preferred path).
   - Wait for probe result. If `status: unhealthy`, surface the error and ask the user to fix (URL, auth header, transport mismatch) before continuing. Do not create agents that reference an unhealthy server.

2. **Build each agent.** For each agent in the plan, in order:
   a. Print: `"About to build agent: <name> (visibility=<v>, tools=<n builtin / m mcp / k function>). Proceed?"`. Wait for `yes`.
   b. Invoke `butterbase-skills:agents` via the Skill tool with: agent name, intended behaviour, plan-derived tool list, model preference, visibility/limits. The wrapped skill drafts the `graph_spec`, writes it to `agents/<name>.json` in the project repo, and calls `validate_agent_spec`.
   c. If validation fails, surface the Zod issues and loop the agents skill to fix.
   d. **Safety gate.** If `visibility != 'private'` AND any node can reach a write tool (`insert_row`, `update_row`, `delete_row`, `write_storage`, or any `read_write`-mode MCP/function tool), explicitly ask: `"This agent is reachable by <visibility> callers and can write data. Set safety_acknowledged=true? (yes / change to private / reduce tools)"`. Do not proceed silently.
   e. `create_agent` with the validated spec.
   f. Smoke: `invoke_agent` with a representative input. Poll `get_agent_run` until terminal. If the run errors, debug per the `agents` skill's debugging procedure; otherwise show the user the final output.
   g. Append one line per agent to `docs/butterbase/04-build-log.md`:
      `<ISO timestamp>  agents  create_agent  <agent-name>  ok (run smoke=<run_id>)`

3. **Persist specs to the repo.** Confirm `agents/<name>.json` files are tracked locally. They will be carried by `butterbase repo push` (run by the `templates` stage if the user is publishing, or any time the user pushes a snapshot). This is the only way a clone recipient can recreate the agents — the `agents` table is **not** part of clone replay.

4. **Tick state.** Mark `- [x] agents` in `00-state.md`, set `current_stage` to the next unchecked stage.

5. Return to `journey` orchestrator (or ask `"Continue to the next stage? (yes/no)"`).

## Outputs

- One or more agents created against the live app.
- One or more `agents/<name>.json` spec files in the project repo.
- One line per agent in `docs/butterbase/04-build-log.md`.

## Anti-patterns

- ❌ Skipping `validate_agent_spec`. Bad specs surface as opaque runtime errors after the agent is live.
- ❌ Creating a `public` write-capable agent without rate limits or a `daily_budget_usd`. The runtime requires `safety_acknowledged`, but you should also pick hourly per-IP and per-app caps.
- ❌ Hard-coding secrets in `system_prompt` or `args_template`. Read them from `ctx.env` inside a function tool.
- ❌ Forgetting that the `agents` table is not replayed on clone. Always commit the spec JSON and document re-import (`butterbase agents create -f agents/<name>.json`) in the README.
- ❌ Smoking with `visibility: 'private'` and calling it good for a `public` agent. After the smoke, change visibility and re-test with an unauthenticated curl to confirm the public path works.
