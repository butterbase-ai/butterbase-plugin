---
name: storage
description: Use when uploading or downloading files, generating presigned URLs, configuring storage ACLs, or persisting file references (avatars, attachments, images) in a Butterbase app
---

# Butterbase Storage

Butterbase stores files in S3 (or LocalStack in dev) and exposes them via presigned URLs. Every file gets a stable `object_id` (UUID) that you persist in your tables; URLs are generated on demand and expire.

All storage operations go through one tool: **`manage_storage`** with an `action` parameter.

| Action | Purpose |
|--------|---------|
| `upload_url` | Generate a 15-minute presigned PUT URL and reserve an `object_id` |
| `download_url` | Generate a 1-hour presigned GET URL for a stored object |
| `list` | List objects (scoped by caller's role) |
| `delete` | Permanently remove an object from S3 + database |
| `update_config` | Toggle app-level `publicReadEnabled` and other storage settings |

---

## 1. The mental model: `object_id` vs `s3_key`

| Field | What it is | When you use it |
|-------|-----------|-----------------|
| `object_id` | UUID, stable, app-level handle | Persist in your tables (e.g. `users.avatar_id`, `posts.image_id`) |
| `s3_key` | Internal bucket path like `app_abc/user_uuid/file.jpg` | Internal only — **never** treat this as a URL |

**Critical:** `s3_key` is **not** a URL. You cannot use it as `<img src>` or `<a href>`. Always store the `object_id` and resolve a fresh download URL at render time.

---

## 2. The upload lifecycle

A single upload is two HTTP calls and one DB insert in your app:

```
┌─────────────────────────┐
│ 1. manage_storage(      │  → returns { upload_url, object_id, expires_at }
│      action: upload_url)│
├─────────────────────────┤
│ 2. PUT file -> S3       │  → must include exact Content-Type header
├─────────────────────────┤
│ 3. INSERT INTO ...      │  → save object_id alongside the user/post/etc.
└─────────────────────────┘
```

If you skip step 3, the file lives in S3 but no row references it — an **orphaned object** counting against your quota. Always persist the `object_id`.

### Step 1 — request an upload URL

```js
manage_storage({
  app_id: "app_abc123",
  action: "upload_url",
  filename: "avatar.jpg",
  content_type: "image/jpeg",
  size_bytes: 245123,
  public: false                  // optional; default false
})
```

Returns:

```json
{
  "upload_url": "https://s3.amazonaws.com/...",
  "object_id": "9c14b2e0-...",
  "expires_at": "2026-05-08T22:15:00Z"
}
```

The `object_id` is created **immediately** in the database; the file just hasn't been uploaded yet.

### Step 2 — PUT the bytes

The Content-Type on the PUT must match exactly the `content_type` you sent in step 1. If it doesn't, S3 rejects the upload or stores the wrong MIME — breaking browser previews.

```bash
curl -X PUT "{upload_url}" \
  -H "Content-Type: image/jpeg" \
  --data-binary @avatar.jpg
```

From the browser:

```js
await fetch(uploadUrl, {
  method: "PUT",
  headers: { "Content-Type": file.type },
  body: file
});
```

### Step 3 — persist the `object_id`

```sql
UPDATE users SET avatar_id = $1 WHERE id = $2
```

Or via the auto-API / SDK — whatever your app uses for writes. Without this step, the file is unreachable.

---

## 3. Generating download URLs

Each call returns a fresh URL valid for 1 hour. Don't bake URLs into static HTML or long-lived caches — re-generate per render or per session.

```js
manage_storage({
  app_id: "app_abc123",
  action: "download_url",
  object_id: "9c14b2e0-..."
})
// → { download_url: "https://s3.amazonaws.com/..." }
```

For lists with many files, resolve URLs in parallel:

```js
const urls = await Promise.all(
  posts.map(p => getDownloadUrl(p.image_id))
);
```

If the caller is unauthorized, the response is `404` (not `403`) — Butterbase deliberately hides existence to avoid leaking object IDs.

---

## 4. Access control

Three tiers, evaluated in order:

| Caller | What they can read |
|--------|--------------------|
| Service key (`bb_sk_*`) | Everything in the app — RLS bypassed |
| End-user JWT | Files where `(user_id === caller_id) OR object.public === true OR app.publicReadEnabled === true` |
| Anonymous (no auth) | Only public objects (and only if app access mode allows anon) |

### Per-object public flag

Set at upload time:

```js
manage_storage({ app_id, action: "upload_url", filename, content_type, size_bytes, public: true })
```

Use this for one-off public files (a marketing image, a shared avatar) without flipping the whole app to public-read.

### App-wide public read

```js
manage_storage({
  app_id: "app_abc123",
  action: "update_config",
  publicReadEnabled: true
})
```

When `true`, any authenticated user in the app can download any file. Uploads and deletes stay user-scoped.

> Storage ACL is hardcoded — you cannot layer Postgres RLS policies on top of `storage_objects`. If you need fine-grained custom rules, gate downloads through a serverless function instead of handing out direct presigned URLs.

---

## 5. Listing and deleting

```js
manage_storage({ app_id: "app_abc123", action: "list" })
```

Service key sees everything; end-user JWT sees only their own files. Each item has `id, user_id, key, filename, content_type, size_bytes, created_at`.

```js
manage_storage({ app_id: "app_abc123", action: "delete", object_id: "9c14b2e0-..." })
```

Permanently removes the S3 object and DB row. **Clear foreign-key references first** (e.g. `UPDATE users SET avatar_id = NULL`) — `manage_storage` doesn't.

---

## 6. Quotas & error codes

| Limit | Default | Override |
|-------|---------|----------|
| Per-file size | 10 MB | `storage_config` |
| Total app storage | Plan-dependent | Upgrade plan |
| Allowed content types | All by default | `storage_config.allowedContentTypes` whitelist |

| Error | When |
|-------|------|
| `QUOTA_FILE_SIZE_EXCEEDED` (400) | `size_bytes` > per-file limit |
| `QUOTA_STORAGE_EXCEEDED` (429) | App total exhausted |
| `VALIDATION_INVALID_TYPE` (400) | `content_type` not in whitelist |
| `RESOURCE_NOT_FOUND` (404) | Object missing or caller unauthorized (deliberately ambiguous) |
| `S3_ERROR` (503) | Transient S3 failure — retry |

---

## 7. Common patterns

### User avatar

1. Upload form posts file → `manage_storage` (`upload_url`, `public: false`).
2. Browser PUTs to `upload_url`.
3. App calls `UPDATE users SET avatar_id = $object_id`.
4. On profile render: `manage_storage` (`download_url`, `object_id: user.avatar_id`).

### Public marketing image

Same flow, but `public: true` at upload time. Then the same download URL is reachable by anonymous visitors (no JWT needed) — useful for landing pages and OG images.

### File attachment with auth

For attachments where access depends on app-level rules richer than "owner or public" (e.g. "members of this workspace can read"), don't expose the presigned URL directly. Wrap downloads in a serverless function:

```ts
export async function handler(req, ctx) {
  const { rows } = await ctx.db.query(
    "SELECT object_id FROM attachments WHERE id = $1 AND workspace_id IN (SELECT workspace_id FROM members WHERE user_id = $2)",
    [attachmentId, ctx.user.id]
  );
  if (!rows.length) return new Response("forbidden", { status: 403 });
  const url = await getDownloadUrlServerSide(rows[0].object_id);
  return Response.redirect(url, 302);
}
```

The function runs as `butterbase_user` and uses your tables' RLS to authorize, then mints a presigned URL with the service key.

---

## 8. Anti-patterns

| Don't | Do |
|-------|-----|
| Store the `s3_key` in your tables | Store the `object_id` UUID |
| Embed presigned URLs in HTML or DB rows | Re-fetch per render — they expire in 1 hour |
| Send the wrong `Content-Type` on PUT | Match exactly what you sent to `upload_url` |
| Call `upload_url` and forget step 3 | Always persist `object_id` after a successful PUT |
| Use `public: true` everywhere "just in case" | Default to private; opt files into public explicitly |
| Delete a file without clearing FKs | Update referencing rows first, then `action: "delete"` |

---

If a `docs/butterbase/00-state.md` exists in the working directory, prefer invoking via `/butterbase-skills:journey-storage` so the journey orchestrator stays in sync.
