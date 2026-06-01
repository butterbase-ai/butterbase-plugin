---
name: migrations
description: Use when moving a Butterbase app between regions, checking migration progress, aborting a stuck move, reverse-rolling a completed move, or tearing down retained source replicas after a move is stable
---

# Butterbase App Migrations (Multi-Region)

Moving an app between regions is an orchestrated migration: data dumped from source, restored in dest, blobs copied, runtime brought up, then traffic flipped. Source replica is retained for safe rollback until you tear it down.

Four small tools wrap the orchestration:

| Tool | Purpose |
|---|---|
| `list_regions` | Discover what regions exist before suggesting one. |
| `move_app` | Start a migration. Returns a `migration_id`. |
| `move_app_status` | Read the current step + replica state of a specific migration. |
| `manage_migrations` | Operational ops: `get_active`, `abort`, `reverse`, `list_source_replicas`. |
| `teardown_source_replica` | Decommission the retained source replica once you're confident. |

---

## 1. The step sequence

A successful move walks through 11 steps in order:

```
requested → reserving_dest → blocking_writes → dumping_data →
restoring_data → copying_blobs → copying_runtime → flipping_routing →
setting_up_reverse_replication → unblocking_writes → completed
```

The cutover happens at `flipping_routing`. Before that, the app is still served from the source. After that, traffic is on the destination.

Terminal states: `completed`, `aborted`, `failed`.

---

## 2. Start a move

```
list_regions              → confirm dest region is supported
move_app(app_id, dest_region) → { migration_id, status: "queued" }
```

Then poll with `move_app_status({ app_id, migration_id })` every few seconds (or call `manage_migrations({ action: "get_active", app_id })` if you've forgotten the id).

---

## 3. Recovery: pick the right escape hatch

Stuck or wrong choice? Different state, different tool:

- **Before `flipping_routing`** (still in dump / restore / copy phases): use `manage_migrations` `action: "abort"`. Cancels cleanly; no data flipped.
- **After `flipping_routing`** (already cut over): abort is locked. Use `manage_migrations` `action: "reverse"`. This rolls back to source — works only while the source replica is still retained (i.e. you haven't called `teardown_source_replica` yet).
- **Move is done and stable** (the new region is serving traffic without issues for a while): use `teardown_source_replica({ migration_id })`. Stops paying for source-region storage. After this, `reverse` is no longer available.

---

## 4. Inspecting in-flight + retained state

- `manage_migrations({ action: "get_active", app_id })` → `{ migration: AppMigration | null }`. Tells you if a move is in progress right now.
- `manage_migrations({ action: "list_source_replicas" })` → all retained replicas the caller owns. Call this before any teardown so you know what you'd lose.

---

## 5. Common pitfalls

- **Calling `move_app` again while one is active** — backend returns 409 `ineligible`. Resolve by waiting for the active migration to terminate, or aborting it first.
- **Trying to abort after cutover** — returns 409. The path forward is `reverse`, not abort.
- **Tearing down a replica too early** — once it's gone, reverse-move is no longer possible. Verify the destination region has been serving traffic cleanly before teardown.
- **Forgetting `dest_region` is validated** — invalid slugs return 400. Always `list_regions` first if you're not sure.

---

## 6. What this skill does NOT cover

- Schema migrations (table/column changes) — use `butterbase-skills:schema-design`.
- Region selection at app *creation* time — use `init_app` directly with the `region` parameter.
- Cross-region replica reads (none — Butterbase is single-region-active per app).
