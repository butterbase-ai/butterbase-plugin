---
name: substrate
description: Use when the user wants to read/write their Butterbase substrate — the per-user agent-memory backend that holds entities, business state, institutional memory, and an append-only action ledger. Use for: founder copilots, AI agents that need memory across sessions, anything that proposes actions on the user's behalf.
---

# Butterbase Substrate

Substrate is an **optional, per-user** add-on backend. One substrate per Butterbase account, lazily provisioned on first use, queryable from local Claude Code (via MCP), from any opted-in Butterbase app (via `ctx.substrate` inside functions), and from external systems (via HTTP).

## Four stores

| Store | What lives there | Tools |
|---|---|---|
| Entities | People, companies, projects you transact with | `get_entity`, `find_entities` |
| Business state | Numeric facts that change over time (MRR, headcount) | `find_entities` with type filter |
| Institutional memory | Decisions, commitments, learnings | `search_memory` |
| Action ledger | Append-only log of every proposed/executed action | `list_outbox`, `propose_action` history |

## The propose → policy → execute → log loop

Every write goes through this loop. Agents NEVER touch the substrate database directly.

1. **Propose** — `propose_action` with `capability` + `args`. Returns an action ID.
2. **Policy** — substrate-core evaluates per-capability rules. Verdicts: `auto_execute`, `require_approval`, `deny`.
3. **Approval (if needed)** — human calls `approve_action(action_id)` or `reject_action(action_id)`. If `yolo_mode=true` for this user, approvals are auto-granted for capabilities marked yolo-safe.
4. **Execute + log** — substrate-core runs the action in a transaction and appends to the ledger.

For external side effects (sending email, calling an API), substrate writes to the **outbox** instead of executing inline. The cron-scheduler drains it. Use `list_outbox`, `retry_outbox`, `cancel_outbox` to manage it.

## Reads

- `get_entity(id)` — fetch by ID.
- `find_entities(type, query, limit)` — typed search; `query` is full-text + structured.
- `search_memory(query, kind?)` — semantic + keyword search across decisions/commitments/learnings.

## When to use

- ✅ Founder copilot ("what did I decide about pricing last quarter?")
- ✅ AI agent that should remember a customer across conversations
- ✅ Any app where you want a single source of truth for entities the agent operates on
- ❌ Plain CRUD app (use regular Butterbase tables)
- ❌ Pure analytics / read-only dashboard (substrate is action-oriented)

## Linking an app

When `apps.substrate_user_id` is set to the app owner's `platform_user.id`, functions in that app get `ctx.substrate` injected at cold start, with reader and proposer methods. Use `/butterbase-skills:journey-substrate` to enable this for an app.

## API keys

For agent access from local Claude Code, generate a `scope='both'` API key via `manage_auth_config` `action: "generate_service_key"` with `substrate_access: true`. The same `bb_sk_` key works on both app and substrate endpoints.

## Anti-patterns

- ❌ Calling substrate MCP tools without first checking whether the user has substrate provisioned. `find_entities` returns 401 if there's no substrate — handle gracefully and suggest provisioning.
- ❌ Treating the action ledger as mutable. It's append-only; "undo" is a new compensating action, not a delete.
- ❌ Storing transient state in entities. Entities are durable nouns; use the app's runtime tables for ephemeral state.
- ❌ Calling `upsert_entity` without `canonical_keys` or `primary_email` and expecting dedup. Provide one or the other so substrate can match an existing row. Without them, every call mints a new entity.
- ❌ Calling `update_entity` for a partial update. It replaces `attrs` wholesale and will drop every key you didn't include. Use `patch_entity` (RFC 7396 merge-patch) instead.
- ❌ Storing entity IDs in app data and skipping alias resolution after `merge_entities`. The merged-away ID stops resolving to an entity; look up `substrate.entity_aliases` to find the survivor.
