---
name: realtime
description: Use when enabling WebSocket subscriptions for live database changes, presence/multiplayer state, or when debugging clients that connect but receive no events
---

# Butterbase Realtime

Live database change notifications over WebSocket, with per-row RLS enforcement. Once a table is enabled, INSERT/UPDATE/DELETE events stream to subscribed clients **filtered by the same RLS policies** that gate reads.

One tool: **`manage_realtime`** with two actions: `configure` and `get`.

---

## 1. The mental model

```
Postgres (data plane)              Control API                       Browser
─────────────────────              ────────────                      ────────
INSERT/UPDATE/DELETE  ──trigger──► realtime.changes  ──WAL listener─► WebSocket ──► client
                                          │
                                          └── RLS check per (role, user) ──► filter rows
```

When you `configure` a table:
1. A Postgres trigger is installed → writes every change to `realtime.changes`.
2. A LISTEN connection in `RealtimeManager` reads those changes.
3. For each connected client, the change is RLS-checked **as that user** before broadcasting.
4. Clients only receive events for rows they could read with a regular SELECT.

---

## 2. Prerequisites

Before calling `configure`, the table must:

1. **Exist.** Run `manage_schema` (`action: "apply"`) first. Realtime won't auto-create it.
2. **Have RLS configured (if you care about isolation).** Realtime respects whatever policies exist via `manage_rls`. **No policies = all events flow to all users of that role.** This is the #1 silent leak.
3. **Have a primary key.** RLS checks query by PK. Tables without one can't be realtime-enabled cleanly.

---

## 3. Configure tables

```js
manage_realtime({
  app_id: "app_abc123",
  action: "configure",
  tables: ["messages", "presence", "documents"]
})
// → [{ table: "messages", status: "enabled" }, ...]
```

- Idempotent — already-enabled tables are skipped.
- All three events (INSERT / UPDATE / DELETE) are enabled together; per-event filtering happens client-side via subscription `filter`.
- Validation: every named table must exist or you get `VALIDATION_TABLE_NOT_FOUND`.

### Inspect current state

```js
manage_realtime({ app_id: "app_abc123", action: "get" })
// → {
//     tables: [{ table_name, enabled, trigger_installed, drift, created_at, updated_at }, ...],
//     active_connection: true,
//     websocket_url: "wss://api.butterbase.dev/v1/app_abc123/realtime"
//   }
```

`drift: true` means the control-plane config says enabled but the data-plane trigger is missing — typically after a schema migration that dropped/recreated the table. Re-run `configure` to repair.

---

## 4. Connect from a client

```
wss://api.butterbase.dev/v1/{app_id}/realtime?token={JWT_or_API_KEY}
```

Browsers can't set custom headers on WebSocket upgrade, so the JWT goes in the query string. Server clients can use `Authorization: Bearer ...` instead.

```js
const ws = new WebSocket(
  `wss://api.butterbase.dev/v1/${appId}/realtime?token=${userJwt}`
);

ws.onopen = () => {
  ws.send(JSON.stringify({ type: "subscribe", table: "messages" }));
};

ws.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  if (msg.type === "change") handleChange(msg);  // { type, table, op, record, old_record, timestamp }
};
```

The Butterbase SDK wraps this:

```ts
const realtime = client.realtime(appId, userJwt);
realtime.subscribe("messages", (change) => console.log(change.op, change.record));
```

### Welcome and protocol

On connect, the server sends:

```json
{ "type": "connected", "app_id": "app_abc123", "role": "butterbase_user" }
```

Then a heartbeat every 30s:

```json
{ "type": "heartbeat", "timestamp": "..." }
```

### Client → server messages

| Type | Body | Purpose |
|------|------|---------|
| `subscribe` | `{ table, filter? }` | Subscribe to changes; optional client-side filter `{ col: value }` |
| `unsubscribe` | `{ table }` | Stop receiving |
| `presence_track` | `{ metadata }` | Announce yourself with arbitrary metadata (cursor, status) |
| `event` | `{ event, payload }` | Trigger a function with `trigger: { type: "websocket", config: { event } }` |

### Server → client messages

| Type | Body |
|------|------|
| `change` | `{ table, op: "INSERT"\|"UPDATE"\|"DELETE", record, old_record, timestamp }` |
| `presence_state` | `{ clients: [{ client_id, user_id, metadata }] }` |
| `heartbeat` | `{ timestamp }` |

---

## 5. RLS enforcement (the critical pitfall)

For each broadcast, the server runs (roughly):

```sql
SET LOCAL ROLE butterbase_user;
SET LOCAL request.jwt.claim.sub = '{user_id}';
SELECT 1 FROM "{table}" WHERE "{pk}" = {record_pk} LIMIT 1;
```

If the row is **not visible** under RLS, the change is **silently dropped** for that client. There is no error.

**Common consequences:**

- Client connects, sees `connected`, subscribes — but receives no events. → RLS too restrictive (or no policies at all + access mode `authenticated`).
- Client receives some events but not others. → RLS works for those rows; others are filtered out (often correct).
- Service key clients see everything. → Service bypasses RLS. Don't use this to "verify realtime works" if testing user-scoped behaviour.

**Always test with a real end-user JWT**, not the service key.

### Anonymous clients

If `manage_app` access mode is `authenticated`, anonymous WebSocket connections are rejected with close code `1008 (Policy Violation)`. To allow anon, the app must be in `public` mode AND the table must have a permissive policy for `butterbase_anon`.

---

## 6. Connection lifecycle

| Close code | Meaning |
|------------|---------|
| `1008` | App requires authentication, no token provided |
| `1013` (try again later) | Plan limit hit (`maxRealtimeListenersPerApp`) — upgrade |
| `1013` ("Realtime disabled by plan") | Free / starter tiers may have realtime off entirely |
| Normal close | Heartbeat missed, client disconnected, or server eviction |

The server caches table primary keys for 60s and batches RLS checks per `(role, user)` group, so connection cost is amortised.

---

## 7. Common patterns

### Live chat

1. `manage_schema` apply with a `messages` table (`id`, `room_id`, `user_id`, `body`, `created_at`).
2. `manage_rls` `create_user_isolation` with `user_column: "user_id"` plus a custom policy that allows reading messages where `room_id IN (SELECT room_id FROM members WHERE user_id = current_user_id())`.
3. `manage_realtime` `configure` with `tables: ["messages"]`.
4. Client connects, subscribes to `messages`, optionally filters `{ room_id: "..." }`.

### Live dashboard / activity feed

1. Add a `notifications` table with `user_id`.
2. RLS `create_user_isolation` so each user only sees their own.
3. Realtime configure → notifications stream straight to the right user, no extra filtering needed.

### Multiplayer cursor sharing

Use **presence**, not table changes:

```js
ws.send(JSON.stringify({
  type: "presence_track",
  metadata: { cursor: { x: 100, y: 50 }, color: "#f00" }
}));
```

Other clients receive `presence_state` updates with everyone's metadata. No DB writes.

### Debugging "client connects but no events"

1. Confirm trigger installed: `manage_realtime` `get` → `trigger_installed: true`, `drift: false`.
2. Confirm RLS allows the user to SELECT the row: `select_rows` with `as_role: "user", as_user: "<id>"` → does the row appear?
3. If yes to both and still no events: check for plan limits in close codes; check that the change *actually* happened in Postgres (look at `realtime.changes`).

---

## 8. Anti-patterns

| Don't | Do |
|-------|----|
| Enable realtime before configuring RLS | Set up policies first; otherwise events leak across users |
| Use a service key from the frontend "to make it work" | Service bypasses RLS — ship-stopping leak. Use end-user JWTs. |
| Trust client-side `filter` for security | `filter` is just a convenience to reduce client-side work; RLS is the security boundary |
| Hold thousands of subscriptions per client | One connection, one or two subscribed tables — the server handles fan-out |
| Re-call `configure` in a loop on every page load | It's idempotent but each call still touches the DB. Configure once during app setup. |
| Send custom auth headers from the browser | WebSocket API can't set them — pass the JWT as `?token=` query param |

---

If a `docs/butterbase/00-state.md` exists in the working directory, prefer invoking via `/butterbase-skills:journey-realtime` so the journey orchestrator stays in sync.
