---
name: deploy-frontend
description: Use when deploying a frontend (React, Next.js, or static HTML) to a live URL on Butterbase, or when troubleshooting deployment issues like MIME type errors or blank pages
---

## Overview

7-step workflow for deploying static frontends to Butterbase. Covers building, CORS, zipping, uploading, and verification.

---

## Framework Reference Table

| Framework       | Build command   | Output dir   | Env prefix      | Framework flag  |
|-----------------|-----------------|--------------|-----------------|-----------------|
| React (Vite)    | `npm run build` | `dist/`      | `VITE_`         | `react-vite`    |
| Next.js (static)| `next build`    | `out/`       | `NEXT_PUBLIC_`  | `nextjs-static` |
| Plain HTML      | (none)          | project root | N/A             | `static`        |

> **Note:** Next.js requires `output: 'export'` in `next.config.js` to produce a static export.

---

## Step 1: Set Environment Variables

Use `manage_frontend` with `action: "set_env"` to configure the API URL and app ID before building. These variables are injected at build time by the framework.

```json
{
  "app_id": "app_abc123",
  "action": "set_env",
  "vars": {
    "VITE_API_URL": "https://api.butterbase.ai/v1/app_abc123",
    "VITE_APP_ID": "app_abc123"
  }
}
```

- For Vite, prefix all public variables with `VITE_`
- For Next.js, prefix with `NEXT_PUBLIC_`
- For Create React App, prefix with `REACT_APP_`

`set_env` upserts; you can call it again to add or change variables.

---

## Step 2: Build

Run the framework-specific build command to produce the static output directory.

| Framework        | Command           |
|------------------|-------------------|
| React (Vite)     | `npm run build`   |
| Next.js (static) | `next build`      |
| Plain HTML       | (no build needed) |

After building, verify the output directory contains `index.html` at its root:

```bash
# For Vite
ls dist/index.html

# For Next.js static export
ls out/index.html
```

If `index.html` is missing, check that the build completed without errors and that the framework is configured for static output.

---

## Step 3: Configure CORS

Before deploying, configure CORS so the browser can make API requests from the deployment URL.

Call `manage_app` with `action: "update_cors"`. Pass the deployment URL (use the Butterbase Pages URL pattern) and any local dev origins:

```json
{
  "app_id": "app_abc123",
  "action": "update_cors",
  "allowed_origins": [
    "https://your-app.pages.dev",
    "http://localhost:5173"
  ]
}
```

- Always include `http://localhost:5173` (Vite dev server default) for local development
- Include `http://localhost:3000` if using Next.js or Create React App locally
- Origins must include the protocol (`https://` or `http://`) and must not have trailing slashes
- If you don't yet know the exact deployment URL, you can update CORS again after Step 7

---

## Step 4: Create Deployment

Call `create_frontend_deployment` with the `app_id` and the correct `framework` flag from the reference table above.

```json
{
  "app_id": "app_abc123",
  "framework": "react-vite"
}
```

The response contains:
- `deployment_id` — save this for Step 7
- `uploadUrl` — the presigned S3 URL for uploading the zip (expires in 15 minutes)

> **Free plan:** 1 deployment per app. Deploying again automatically replaces the previous deployment — no need to delete first.

---

## Step 5: Create Zip

> ⚠️ **CRITICAL Windows Warning:** Zip files MUST use forward slashes (`/`). On Windows, built-in zip tools (File Explorer, PowerShell `Compress-Archive`) use backslashes (`\`), which causes **ALL files to be served as `text/html`** — breaking JS and CSS with MIME type errors. Always use Git Bash or WSL on Windows.

**Windows (Git Bash or WSL):**
```bash
cd dist && zip -r ../frontend.zip .
```

**Mac/Linux:**
```bash
cd dist && zip -r ../frontend.zip .
```

> **Important:** Always zip from **inside** the output directory so `index.html` is at the root of the zip, not nested inside a subdirectory. If you zip from outside (e.g., `zip -r frontend.zip dist/`), the zip will contain a `dist/` folder, and the deployment will show a blank page.

For Next.js static exports, replace `dist` with `out`:
```bash
cd out && zip -r ../frontend.zip .
```

---

## Step 6: Upload

Upload the zip file to the presigned S3 URL returned in Step 4:

```bash
curl -X PUT "{uploadUrl}" \
  -H "Content-Type: application/zip" \
  --data-binary @frontend.zip
```

- Replace `{uploadUrl}` with the full presigned URL from Step 4
- The upload URL expires in **15 minutes** — if it expires, repeat Step 4 to get a new one
- Maximum file size: **100 MB**
- A successful upload returns an empty 200 response with no body

---

## Step 7: Start & Verify

Call `manage_frontend` with `action: "start_deployment"` and the `deployment_id` from Step 4:

```json
{
  "app_id": "app_abc123",
  "action": "start_deployment",
  "deployment_id": "uuid-1234"
}
```

- The tool polls until the deployment status is `READY` (up to 5 minutes)
- On success, it returns the live URL (e.g., `https://your-app.pages.dev`)

**Verification checklist:**
1. Open the live URL in a browser
2. Check the browser console (F12) for JavaScript errors or failed network requests
3. Navigate to a non-root route to verify SPA routing works (auto-handled for `react-vite` and `nextjs-static`)
4. Make an API call and confirm it succeeds (no CORS errors)

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Blank page | `index.html` not at zip root | Re-zip from inside output dir: `cd dist && zip -r ../frontend.zip .` |
| MIME type errors / broken JS/CSS | Windows backslash in zip paths | Re-zip using Git Bash or WSL: `cd dist && zip -r ../frontend.zip .` |
| API calls return 403 | CORS not configured | Add deployment URL via `manage_app` action `update_cors` |
| Routes return 404 | SPA routing not set up | SPA routing is auto-handled for `react-vite` and `nextjs-static` framework flags |
| Deploy stuck in BUILDING | Build error | Check `manage_frontend` action `list_deployments` for `error` field |
| Upload fails or curl errors | Upload URL expired | Get a new URL by calling `create_frontend_deployment` again |
| Next.js pages not exporting | Missing static export config | Add `output: 'export'` to `next.config.js` and rebuild |
| Environment variables not found | Not set before build | Run `manage_frontend` action `set_env` and rebuild — env vars are baked in at build time |

---

## Bonus: server-side build (no local build needed)

If you want Butterbase to run `npm install` + build on the server, skip Steps 2–6 and use the source-build flow:

1. `manage_frontend` action `create_from_source` → returns presigned URL for a source-code zip (≤ 50 MB)
2. `cd <project> && zip -r ../source.zip . -x "node_modules/*" ".next/*" "dist/*"` — exclude build artefacts
3. `curl -X PUT "{uploadUrl}" -H "Content-Type: application/zip" --data-binary @source.zip`
4. `manage_frontend` action `start_from_source` with `deployment_id`, `lockfile_hash` (sha256 of `package-lock.json`), optional `build_command`, `output_dir`, `package_manager`, `user_env`

Same `lockfile_hash` across deploys = cached `node_modules` for faster rebuilds.

---

## Edge SSR (Next.js / Remix with server-side rendering)

Static deploys can't run server-side code. For Next.js (App Router with SSR/middleware) or Remix on Cloudflare Workers, use `manage_edge_ssr` instead of `create_frontend_deployment` + `manage_frontend`. Same shape, separate tool:

| Action | Purpose |
|--------|---------|
| `create` | Returns presigned URL for a pre-built Workers zip (framework default `nextjs-edge`) |
| `start` | Uploads + deploys; polls ≤ 60s |
| `create_from_source` / `start_from_source` | Server-side build flow (defaults: `npx @cloudflare/next-on-pages`, output `.vercel/output/static`) |
| `list` | Historical edge deployments |

Frameworks: `nextjs-edge`, `remix-edge`, `other-edge`.
