---
name: auth-setup
description: Use when configuring OAuth providers (Google/GitHub/Apple/X/etc.), setting up post-login auth hooks, tuning JWT lifetimes, or generating service API keys
---

# Butterbase Auth Setup

Two umbrella tools cover end-user authentication:

- **`manage_oauth`** — provider configuration (Google, GitHub, Apple, X, custom)
- **`manage_auth_config`** — auth hooks, JWT lifetimes, service key generation

For broad app build-out, see also `butterbase-skills:build-app`. This skill is the deep dive.

---

## 1. The role model

Every request runs under one of three database roles:

| Auth header | Role | `current_user_id()` | RLS |
|-------------|------|---------------------|-----|
| _none_ | `butterbase_anon` | NULL | enforced; default deny |
| End-user JWT (issued by `manage_oauth` or email login) | `butterbase_user` | user UUID | enforced |
| Service key (`bb_sk_*`) | `butterbase_service` | NULL | bypassed |

Auth is what transforms a request into the right role. RLS is what filters the data. Both must be configured.

---

## 2. Configure an OAuth provider

```js
manage_oauth({
  app_id: "app_abc123",
  action: "configure",
  provider: "google",
  client_id: "123456789.apps.googleusercontent.com",
  client_secret: "GOCSPX-...",
  redirect_uris: ["https://api.butterbase.ai/auth/app_abc123/oauth/google/callback"]
  // scopes / authorization_url / token_url / userinfo_url / provider_metadata are auto-filled for built-in providers
})
```

**Built-in providers (URLs and scopes pre-filled):** `google`, `github`, `discord`, `facebook`, `linkedin`, `microsoft`, `apple`, `x`.

**Custom providers:** pass `authorization_url`, `token_url`, `userinfo_url`, and `scopes` explicitly.

### Redirect URI format

```
https://api.butterbase.ai/auth/{app_id}/oauth/{provider}/callback
```

Register **this exact URI** in the provider's developer console. Mismatch is the most common reason OAuth flows fail.

### Provider quirks

| Provider | Quirk |
|----------|-------|
| `apple` | Requires `provider_metadata: { teamId, keyId, privateKey }`. Apple only returns the user's name on **first** auth and uses POST callback (handled automatically). |
| `x` | Does not return email. Butterbase synthesises `{username}@users.noreply.x.local` for the user record. |
| `facebook` | Default scopes `email`, `public_profile`. |
| `google` | Standard. |
| `github` | Standard. |

### List, update, delete

```js
manage_oauth({ app_id, action: "get" })                          // list all providers (secrets redacted)
manage_oauth({ app_id, action: "get", provider: "google" })      // single provider
manage_oauth({ app_id, action: "update", provider: "google", client_secret: "new-secret" })
manage_oauth({ app_id, action: "delete", provider: "google" })   // disables future logins; existing sessions valid until expiry
```

### Frontend flow

```
GET https://api.butterbase.ai/auth/{app_id}/oauth/{provider}?redirect_to=https://yourapp.com/auth/callback
```

User signs in at the provider, gets bounced back to `redirect_to` with `access_token` and `refresh_token` as query params. The Butterbase SDK wraps this:

```ts
await client.auth.signInWithOAuth({ provider: "google" });
const { user, accessToken } = await client.auth.getSession();
```

---

## 3. Tune JWT lifetimes

```js
manage_auth_config({
  app_id: "app_abc123",
  action: "update_jwt",
  accessTokenTtl: "15m",       // formats: "15m", "1h", "2h", "1d"
  refreshTokenTtlDays: 30      // integer days
})
```

Defaults: 15-minute access tokens, 7-day refresh tokens.

| Use case | `accessTokenTtl` | `refreshTokenTtlDays` |
|----------|------------------|------------------------|
| High-security (banking, admin) | `5m`–`15m` | `1`–`7` |
| Standard SaaS | `15m` (default) | `30` |
| Low-friction consumer apps | `1h` | `90` |

**Important:** changes apply only to **new** tokens. Active tokens keep their original expiration — there is no global revoke. Treat TTL changes as forward-looking only.

---

## 4. Auth hooks (run code after every login)

A post-auth function is a deployed Butterbase function invoked **fire-and-forget** after every successful auth event (OAuth login, email login, email signup).

### Wire it up

```js
// 1. Deploy the function first (see butterbase-skills:function-dev)
deploy_function({
  app_id: "app_abc123",
  name: "after-auth",
  code: postAuthHandlerCode,
  trigger: { type: "http", config: { auth: "none" } }
})

// 2. Register it as the auth hook
manage_auth_config({
  app_id: "app_abc123",
  action: "configure_auth_hook",
  post_auth_function: "after-auth"
})

// To remove the hook later: pass post_auth_function: null
```

The function **must already exist** when you configure the hook.

### Payload shape

The function receives a POST with this body:

```json
{
  "event": "oauth_login | login | signup",
  "user": {
    "id": "uuid",
    "email": "...",
    "provider": "google | github | email | ...",
    "display_name": "...",
    "avatar_url": "..."
  },
  "isNewUser": true,
  "provider": "google"
}
```

The function runs as `butterbase_service` (RLS bypassed, `ctx.user` is `null`). Use `body.user.id` to know who just logged in.

### Common uses

```ts
// after-auth/index.ts
export async function handler(req, ctx) {
  const { user, isNewUser, event } = await req.json();

  if (isNewUser) {
    // 1. Create profile row
    await ctx.db.query(
      "INSERT INTO profiles (user_id, display_name) VALUES ($1, $2) ON CONFLICT DO NOTHING",
      [user.id, user.display_name]
    );

    // 2. Send welcome email (via env-stored API key)
    ctx.waitUntil(sendWelcomeEmail(ctx.env.RESEND_API_KEY, user.email));
  }

  // 3. Audit log on every login
  await ctx.db.query(
    "INSERT INTO login_log (user_id, event, provider) VALUES ($1, $2, $3)",
    [user.id, event, ctx.user ?? null]
  );

  return new Response("ok", { status: 200 });
}
```

> Auth hooks are fire-and-forget. Don't return data the user needs — they won't see it. Use them for side effects only.

---

## 5. Service keys (`bb_sk_*`)

Service keys grant **full access** to all your apps and bypass RLS. Treat them like passwords.

### Generate

```js
manage_auth_config({
  action: "generate_service_key",
  name: "CI/CD pipeline"
})
// → { key: "bb_sk_a1b2c3...", key_id, prefix, name, created_at }
```

> The full key is returned **once**. Store it immediately in your secret manager — you cannot retrieve it again. If you lose it, generate a new one and revoke the old.

### List & revoke

```js
manage_api_keys({ action: "list" })
manage_api_keys({ action: "revoke", key_id: "uuid-..." })
```

`list` returns metadata only (prefix, name, last_used_at), never the secret. `revoke` is **immediate and irreversible**.

### Rotation workflow

1. `manage_auth_config` (`generate_service_key`) — create the new key.
2. Update CI/CD, MCP config, scripts to use the new key.
3. Verify with a smoke test (e.g. `manage_app` `list`).
4. `manage_api_keys` (`revoke`) — kill the old key.

Do steps 1–3 **before** step 4 to avoid downtime.

---

## 6. Anti-patterns

| Don't | Do |
|-------|----|
| Hardcode `bb_sk_*` keys in client code or commit them to git | Store in env vars / secret manager |
| Reuse one OAuth app between dev, staging, prod | Separate OAuth apps per environment, with their own redirect URIs |
| Use a service key from frontend code "for convenience" | Frontends use end-user JWTs; service keys are server-only |
| Increase `accessTokenTtl` to "fix" frequent re-auth | Use the refresh token; SDK handles this automatically |
| Forget that `manage_oauth` `delete` only stops *new* logins | Existing sessions remain valid until they expire — rotate JWT keys via support if you need a hard kill |
| Put critical logic in the auth hook | Hooks are fire-and-forget. Errors don't surface to the user. Keep them to side effects. |
| Log the full service key in audit / debug output | Log only the prefix (`bb_sk_a1b2c3`) — secrets must never appear in logs |

---

## 7. Quick reference

| Task | Tool |
|------|------|
| Add Google OAuth | `manage_oauth` (`configure`) |
| List OAuth providers | `manage_oauth` (`get`) |
| Set post-login hook | `manage_auth_config` (`configure_auth_hook`) |
| Change JWT lifetimes | `manage_auth_config` (`update_jwt`) |
| Create service key | `manage_auth_config` (`generate_service_key`) |
| List service keys | `manage_api_keys` (`list`) |
| Revoke service key | `manage_api_keys` (`revoke`) |

---

If a `docs/butterbase/00-state.md` exists in the working directory, prefer invoking via `/butterbase-skills:journey-auth` so the journey orchestrator stays in sync.
