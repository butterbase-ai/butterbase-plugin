---
name: agents
description: Use when designing, deploying, or debugging a Butterbase Agent (declarative LLM/tool graph), registering an MCP server for tool use, or wiring access controls and rate limits. Agents are first-class app resources defined by a `graph_spec` and invoked over `/v1/<app_id>/agents/<name>/runs`.
---

# Butterbase Agents

A Butterbase agent is a **declarative graph** of LLM and tool nodes — not a free-running chat loop. The runtime traverses the graph, calls tools (builtin / MCP / function), and resolves the `end` node's `output_template`. State, rate limits, and budgets are enforced by the control plane.

## When to use

- The user wants to add a workflow that combines an LLM with tool calls (DB writes, storage reads, MCP servers, app functions).
- The user wants to expose an agent endpoint to end users (`visibility: public` or `authenticated`).
- Debugging a failing agent run (look at `list_agent_runs`, then `get_agent_run`).
- Registering an external MCP server for the agent to use.

Don't use for plain LLM chat completions — use the `ai` skill (`manage_ai` / `/v1/ai/chat`). Agents are for stateful, multi-step, tool-using workflows.

## Concepts

### `graph_spec` (validated by `validate_agent_spec` before anything is persisted)

| Field | Required | Notes |
|---|---|---|
| `spec_version` | yes | Literal `"1"`. |
| `entry` | yes | ID of the first node. |
| `nodes` | yes | Record `{ id → node }`. |
| `edges` | yes | `[{ from, to }]`. Both endpoints must exist in `nodes`. |
| `tools` | yes | `{ builtin: [], mcp_servers: [], functions: [] }` — declares what nodes can call. |
| `limits` | yes | `max_steps` (1–200), `max_tool_calls` (0–500), `max_parallel_tools` (1–16), `timeout_seconds` (5–3600), `human_timeout_seconds` (60–7×24×3600). |

**Node types:**

- **`llm`** — `model`, `system_prompt`, `input_template`, `output_key`, `tools: [toolRef]`, optional `temperature` (0–2), `max_tokens`.
- **`tool`** — `tool_ref`, `args_template` (record), `output_key`.
- **`end`** — `output_template` (string; can interpolate `{{output_key}}` values).

**`toolRef`** is a discriminated union by `source`:
- `{ source: 'builtin', name }`
- `{ source: 'mcp', server_id, name }`
- `{ source: 'function', name }`

Each may carry `mode_override` (`read_only` | `read_write`) and `exposed_to_override` (`developer_only` | `end_user`).

### Builtin tools (always available, no setup)

| Name | Purpose | Args |
|---|---|---|
| `query_table` | Select rows (RLS enforced) | `table`, `filter`, `limit` (≤200) |
| `insert_row` | Insert | `table`, `values` |
| `update_row` | Update by id | `table`, `id`, `patch` |
| `delete_row` | Delete by id | `table`, `id` |
| `read_storage` | Get object (≤5 MB) | `key` |
| `write_storage` | Put object (≤1 MB b64) | `key`, `content_base64`, `content_type?` |
| `auth_user_lookup` | Find a user | `email` OR `id` |

All builtins respect role: `end_user` runs as `butterbase_user` with their user id (RLS applies); `developer_only` runs as `butterbase_service`.

### MCP servers

Register before referencing in `graph_spec.tools.mcp_servers`. Transports: `sse`, `http`, `streamable_http`. The control plane **probes** on register (calls `listTools()`), stores `status='healthy'|'unhealthy'`. Re-probe with the same endpoint after a server URL change.

### Access & limits

| Field | Default | Notes |
|---|---|---|
| `visibility` | `private` | `private` (owner only), `authenticated` (any app user), `public` (anyone, with rate limits). |
| `max_runs_per_user_per_hour` | null | null = unlimited. |
| `max_runs_per_ip_per_hour` | null | Primary public-agent throttle. |
| `max_runs_per_app_per_hour` | null | App-wide cap. |
| `daily_budget_usd` | null | Hard kill once exceeded. |
| `max_concurrent_runs` | null | |
| `safety_acknowledged` | false | **Required true** if visibility ≠ private AND any node calls a write tool (`insert_row`, `update_row`, `delete_row`, `write_storage`, or a write-mode MCP/function tool). |

## Procedure

### Designing a new agent

1. **Sketch the graph in prose first.** "User asks X → LLM rephrases → query_table for context → LLM answers → end." Concrete node IDs.
2. **Write the spec** as a JSON file in the repo (e.g. `agents/<name>.json`) — versioning it in git makes templates portable and lets `butterbase repo push` carry it to clones.
3. **Validate without persisting** — call `validate_agent_spec` (MCP) or pass the file to a `validate_agent_spec` call. Surface any Zod issues to the user with field paths.
4. **Register MCP servers** if used: `agent_mcp_servers` table (MCP-tool wrapper TBD; use the dashboard or POST `/v1/<app_id>/agent-mcp-servers` directly). Wait for `status: healthy`.
5. **Create** — `create_agent` with name, graph_spec, default_model, access fields. If `visibility ≠ 'private'` and any write tool is reachable, require the user to explicitly say "yes, I acknowledge" and set `safety_acknowledged: true`.
6. **Smoke** — `invoke_agent` with a small input. Poll `get_agent_run` until terminal. Show the user the run timeline (steps, tool calls, final output).

### Editing

- `update_agent` is a PATCH. Pass only changed fields. Bumping `graph_spec` revalidates; runs in flight against the old spec finish unmolested.
- Disabling an agent: `update_agent { status: 'disabled' }` — new runs return 403, existing runs keep going.

### Debugging a failing run

1. `list_agent_runs` filtered by agent name, then `get_agent_run(run_id)` for the event timeline.
2. Check `error.code`: `validation_failed` (spec issue), `tool_error` (named tool, named arg), `budget_exceeded`, `rate_limited`, `timeout`.
3. For tool errors, re-run the same `args_template` with the underlying tool directly (`select_rows`, `invoke_function`, etc.) to confirm the issue is in the tool's surface, not the agent runtime.
4. For `human_input_required` checkpoints, resume with `resume_agent_run(run_id, user_input)`.

### CLI

- `butterbase agents list` / `get <name>` / `create -f spec.json` / `update <name> -f patch.json` / `delete <name>` — read/write specs from files. Useful for version-controlling agents alongside app code.

## Anti-patterns

- ❌ Skipping `validate_agent_spec`. Zod issues are clearer than the runtime errors you get from a bad spec at first invocation.
- ❌ Setting `visibility: public` with write tools and no rate limits. The control plane will refuse without `safety_acknowledged: true`, but you should also set per-IP limits and a daily budget.
- ❌ Putting secrets in `system_prompt` or `args_template`. Read them from `ctx.env` inside a function tool instead — agent specs are visible to anyone who can read the agent.
- ❌ Letting an LLM node call itself recursively without a `max_steps` ceiling. Always cap.
- ❌ Forgetting that builtin DB tools respect RLS. If `query_table` returns empty, the calling role probably can't see the rows — check `exposed_to`.
- ❌ Treating agents as part of clone replay. Agent records are **not** copied when an app is cloned — bundle the spec JSON in the repo (`agents/*.json`) and document recreation in the README.
