---
name: durable-objects
description: Use when building stateful per-key actors — chat rooms, multiplayer rooms, rate limiters, long-running agents, leaderboards — that need persistent in-memory + storage state across requests
---

# Butterbase Durable Objects

Durable Objects (DOs) are **stateful per-key actors** running on Cloudflare Workers. Each instance has its own in-memory state and a built-in transactional KV store. Use one when state must survive across requests for a single room/user/agent. For stateless work, use a serverless function instead (`butterbase-skills:function-dev`).

One tool: **`manage_durable_objects`**.

---

## 1. The mental model

```
Class: ChatRoom (deployed once)
   │
   ├── instance "lobby"      ─►  in-memory state + state.storage  +  WebSockets
   ├── instance "general"    ─►  separate state, separate sockets
   └── instance "user-123"   ─►  separate again

Each URL  https://<app>.butterbase.dev/_do/chat-room/<instance-id>
gets routed to the instance with that id. State is isolated per id.
```

A class is shared code; an **instance** is a unique key (`/lobby`, `/general`, `/user-123`). Different ids = different state. There is no shared cross-instance state.

---

## 2. Constraints (read these first)

- **One TypeScript file per class.** No npm imports. Only `import { ... } from 'cloudflare:workers'` is allowed.
- **Exactly one exported class.** `export class Foo { ... }` — no extra exports, no helpers re-exported.
- **PascalCase class name** in source; **kebab-case** for the URL name (e.g. `ChatRoom` ↔ `chat-room`).
- File size: ≤ 5 MB. Total of all DO classes per app: ≤ 10 MB compressed.
- ≤ 5 DO classes per app (v1).
- **No service bindings yet.** Functions reach DOs over HTTP, not via env binding.
- `state.storage` keys/values capped at 128 KB. Larger blobs → Butterbase Storage.
- **WebSockets need `access_mode: "public"`** because browsers can't send custom headers on WS upgrade. Validate auth tokens inside `fetch()` instead.

---

## 3. The class skeleton

```typescript
export class ChatRoom {
  constructor(public state: DurableObjectState, public env: Env) {}

  async fetch(req: Request): Promise<Response> {
    if (req.headers.get("Upgrade") === "websocket") {
      const pair = new WebSocketPair();
      this.state.acceptWebSocket(pair[1]);
      return new Response(null, { status: 101, webSocket: pair[0] });
    }

    if (req.method === "POST") {
      // handle plain HTTP
    }

    return Response.json({ ok: true });
  }

  // Optional WebSocket lifecycle hooks — called by the runtime
  async webSocketMessage(ws: WebSocket, msg: string | ArrayBuffer) {
    if (typeof msg !== "string") return;             // guard binary
    for (const peer of this.state.getWebSockets()) {
      try { peer.send(msg); } catch {}
    }
  }

  async webSocketClose(ws: WebSocket, code: number, reason: string, wasClean: boolean) {}
  async webSocketError(ws: WebSocket, err: Error) {}
}
```

Key APIs:

| API | Purpose |
|-----|---------|
| `state.storage.get/put/delete/deleteAll/list` | Async transactional KV store |
| `state.acceptWebSocket(ws)` | Hold a WS connection; runtime routes messages to `webSocketMessage` |
| `state.getWebSockets()` | All active WS connections for this instance |
| `new WebSocketPair()` | Returns `[client, server]` — return `client` to browser, accept `server` |
| `this.env.KEY` | Read DO env vars (set via `set_env`) |

---

## 4. Deploy

```js
manage_durable_objects({
  app_id: "app_abc123",
  action: "deploy",
  name: "chat-room",                  // kebab-case URL name
  code: "<single TypeScript file>",
  access_mode: "authenticated"        // "public" | "authenticated" (default) | "service_key"
})
// → { id, name, class_name, status: "READY", access_mode, last_deployed_at }
```

Re-deploying with the same `name` updates the class; old in-memory state is evicted on next request. Storage persists across redeploys (same instance id = same `state.storage`).

### Access modes

| Mode | Auth required |
|------|---------------|
| `public` | None — validate tokens inside `fetch()` if you need any |
| `authenticated` (default) | End-user JWT in `Authorization: Bearer <token>` |
| `service_key` | Butterbase service key — backend-to-backend |

> The dispatcher only checks header **shape**, not validity. For real auth on production DOs, validate the token inside `fetch()`.

---

## 5. Address an instance

```
https://<your-subdomain>.butterbase.dev/_do/<name>/<instance-id>
```

- `<name>` = kebab-case DO name from deploy
- `<instance-id>` = anything you choose (`/lobby`, `/user-123`, `/main`)

Both HTTP and WebSocket upgrade work on the same URL.

```js
// HTTP
fetch("https://app.butterbase.dev/_do/chat-room/lobby", {
  method: "POST",
  body: JSON.stringify({ user: "alice", text: "hi" })
});

// WebSocket
const ws = new WebSocket("wss://app.butterbase.dev/_do/chat-room/lobby");
```

Different instance ids → completely separate state. There is no shared global view; if you need one, build it yourself (e.g. a `/registry` instance that other instances report into).

---

## 6. Env vars

Env vars are app-wide across all DO classes. Setting one redeploys the DO Worker — existing in-memory state is evicted, active WS connections drop.

```js
manage_durable_objects({ app_id, action: "list_env" })                                  // keys only, never values
manage_durable_objects({ app_id, action: "set_env", key: "AI_API_KEY", value: "sk-..." })
manage_durable_objects({ app_id, action: "delete_env", key: "AI_API_KEY" })
```

- Keys must match `^[A-Z_][A-Z0-9_]*$` (UPPER_SNAKE).
- A key can't collide with a DO class binding (e.g. `chat-room` reserves `CHAT_ROOM`).
- Read in code as `this.env.KEY_NAME`.

---

## 7. Lifecycle, listing, deletion

```js
manage_durable_objects({ app_id, action: "list" })
manage_durable_objects({ app_id, action: "get", name: "chat-room" })             // includes full source + status + error_message
manage_durable_objects({ app_id, action: "delete", name: "chat-room" })          // IRREVERSIBLE: purges all instances + storage
manage_durable_objects({ app_id, action: "usage", name: "chat-room" })           // do_requests, do_cpu_ms (refreshed every 15 min)
```

Status transitions: `PENDING → BUILDING → READY` or `ERROR` (with `error_message`).

---

## 8. Patterns

### Chat room (broadcast)

```typescript
export class ChatRoom {
  constructor(public state: DurableObjectState, public env: any) {}

  async fetch(req: Request): Promise<Response> {
    if (req.headers.get("Upgrade") === "websocket") {
      const pair = new WebSocketPair();
      this.state.acceptWebSocket(pair[1]);
      const history = (await this.state.storage.get("messages")) ?? [];
      pair[1].send(JSON.stringify({ type: "init", messages: history }));
      return new Response(null, { status: 101, webSocket: pair[0] });
    }
    return Response.json(await this.state.storage.get("messages") ?? []);
  }

  async webSocketMessage(ws: WebSocket, msg: string | ArrayBuffer) {
    if (typeof msg !== "string") return;
    const history: any[] = (await this.state.storage.get("messages")) ?? [];
    const parsed = JSON.parse(msg);
    history.push(parsed);
    await this.state.storage.put("messages", history.slice(-200));
    for (const peer of this.state.getWebSockets()) {
      try { peer.send(msg); } catch {}
    }
  }
}
```

### Sliding-window rate limiter

```typescript
export class RateLimiter {
  constructor(public state: DurableObjectState, public env: any) {}

  async fetch(req: Request): Promise<Response> {
    const now = Date.now();
    const window = 60_000;
    const limit = 100;
    const requests: number[] = (await this.state.storage.get("requests")) ?? [];
    const recent = requests.filter(t => now - t < window);
    if (recent.length >= limit) return new Response("rate limit", { status: 429 });
    recent.push(now);
    await this.state.storage.put("requests", recent);
    return Response.json({ ok: true, remaining: limit - recent.length });
  }
}
```

Address one instance per actor: `/_do/rate-limiter/<user-id>` or `/_do/rate-limiter/<api-key-hash>`.

### Long-running AI agent

```typescript
export class Agent {
  constructor(public state: DurableObjectState, public env: any) {}
  async fetch(req: Request): Promise<Response> {
    const { prompt } = await req.json();
    const r = await fetch(this.env.AI_API_ENDPOINT, {
      method: "POST",
      headers: { Authorization: `Bearer ${this.env.AI_API_KEY}` },
      body: JSON.stringify({ prompt })
    });
    const data = await r.json();
    await this.state.storage.put("last_response", data);
    return Response.json(data);
  }
}
```

One DO instance per conversation; storage holds the rolling history.

### Counter / leaderboard

Each instance is its own counter. `/_do/leaderboard/main` and `/_do/leaderboard/season-2` have independent state — no coordination needed in v1.

---

## 9. Errors

Build-time (rejected on deploy):

| Code | Cause |
|------|-------|
| `NO_EXPORTED_CLASS` | Source doesn't export a class |
| `MULTIPLE_EXPORTS` | More than one export, or export of non-class |
| `INVALID_IMPORT` | Imported anything other than `cloudflare:workers` |
| `CLASS_NAME_PARSE_ERROR` | TS AST couldn't extract the class name |
| `NAME_REGEX_VIOLATION` | `name` doesn't match `^[a-z][a-z0-9]*(?:-[a-z0-9]+)*$` |
| `QUOTA_DO_LIMIT` | Already 5 classes for this app |
| `BUNDLE_SIZE_EXCEEDED` / `SOURCE_SIZE_EXCEEDED` | Over the 10 MB / 5 MB limits |

Runtime / async-deploy:

- Status `ERROR` after a deploy → check `manage_durable_objects` (`get`) `error_message`.
- WebSocket message handler only fires if you called `state.acceptWebSocket(ws)` — easy to forget.
- `webSocketMessage` receives `string | ArrayBuffer`. Always guard before `JSON.parse`.

---

## 10. Anti-patterns

| Don't | Do |
|-------|----|
| Try to share state between instances directly | Pick a single "registry" instance and have others fetch into it |
| Use a DO for stateless HTTP work | Use a function — DOs cost more and have stricter constraints |
| Rely on dispatcher access_mode for real auth | Validate JWTs inside `fetch()` for production |
| Use `access_mode: "authenticated"` for browser WebSockets | Use `public` + token-in-query-string + manual validation; browsers can't set headers |
| Stuff > 128 KB blobs into `state.storage` | Use Butterbase Storage and store the `object_id` in DO state |
| Update env vars in tight loops | Each `set_env` redeploys the Worker — drops connections |
| Forget redeploy semantics | Code change or env change evicts all instances; storage survives but in-memory caches don't |

---

If a `docs/butterbase/00-state.md` exists in the working directory, prefer invoking via `/butterbase-skills:journey-durable` so the journey orchestrator stays in sync.
