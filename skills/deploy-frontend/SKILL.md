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

## Step 5: Create Zip (Node `archiver` — the only supported method)

> ⚠️ **Do not use `Compress-Archive`, File Explorer, or `zip -r` from outside the build dir.** Windows built-in tools write backslash (`\`) path separators, which makes the platform serve every file as `text/html` and breaks JS/CSS with MIME errors. Zipping from the parent dir nests `dist/` inside the archive and ships a blank page.

Butterbase's recommended cross-platform method is the [`archiver`](https://www.npmjs.com/package/archiver) Node package. It always writes POSIX `/` separators (works identically on macOS, Linux, Windows PowerShell, cmd, Git Bash, WSL) and zips from *inside* the source dir so `index.html` lands at the zip root.

**One-time setup in the project being deployed:**

```bash
npm install --save-dev archiver
mkdir -p scripts
```

**Then save this as `scripts/make-zip.mjs` (copy verbatim):**

```js
#!/usr/bin/env node
/**
 * Butterbase frontend zipper — the only supported way to compress a build
 * for `create_frontend_deployment` / `create_from_source`.
 *
 * Usage:
 *   node scripts/make-zip.mjs <sourceDir> <outZip> [--exclude=glob,glob,...]
 *
 * Examples:
 *   node scripts/make-zip.mjs dist frontend.zip                # Vite
 *   node scripts/make-zip.mjs out  frontend.zip                # Next.js static export
 *   node scripts/make-zip.mjs .    source.zip \                # source-build flow
 *     --exclude=node_modules,.next,dist,out,.git,.turbo,.cache
 */
import { createWriteStream } from "node:fs";
import { stat } from "node:fs/promises";
import { resolve } from "node:path";
import archiver from "archiver";

const [, , srcArg, outArg, ...rest] = process.argv;
if (!srcArg || !outArg) {
  console.error(
    "usage: node make-zip.mjs <sourceDir> <outZip> [--exclude=glob,glob,...]"
  );
  process.exit(2);
}

const src = resolve(srcArg);
const out = resolve(outArg);

const excludeFlag = rest.find((a) => a.startsWith("--exclude="));
const excludes = excludeFlag
  ? excludeFlag
      .slice("--exclude=".length)
      .split(",")
      .map((s) => s.trim())
      .filter(Boolean)
      .flatMap((g) => [g, `${g}/**`])
  : [];

const srcStat = await stat(src).catch(() => null);
if (!srcStat?.isDirectory()) {
  console.error(`error: source is not a directory: ${src}`);
  process.exit(1);
}

const output = createWriteStream(out);
const archive = archiver("zip", { zlib: { level: 9 }, forceLocalTime: true });

output.on("close", () => {
  const mb = (archive.pointer() / (1024 * 1024)).toFixed(2);
  console.log(`wrote ${out} (${mb} MB, ${archive.pointer()} bytes)`);
});
archive.on("warning", (err) => {
  if (err.code === "ENOENT") console.warn(err);
  else throw err;
});
archive.on("error", (err) => {
  throw err;
});

archive.pipe(output);
// cwd: src + glob('**/*') ⇒ entries are relative to src, so index.html
// sits at the zip root. archiver normalises separators to '/' on every OS.
archive.glob("**/*", { cwd: src, dot: true, ignore: excludes });
await archive.finalize();
```

**Run it:**

| Framework        | Command                                          |
|------------------|--------------------------------------------------|
| Vite             | `node scripts/make-zip.mjs dist frontend.zip`    |
| Next.js (static) | `node scripts/make-zip.mjs out  frontend.zip`    |
| Plain HTML       | `node scripts/make-zip.mjs .    frontend.zip --exclude=node_modules,.git` |

The script prints the final size on success — must be ≤ 100 MB for static deploys, ≤ 50 MB for source-build.

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
| Blank page | `index.html` not at zip root | Re-zip with the supported script: `node make-zip.mjs dist frontend.zip` |
| MIME type errors / broken JS/CSS | Windows backslash in zip paths (Compress-Archive, File Explorer) | Re-zip with `node make-zip.mjs dist frontend.zip` — archiver writes POSIX `/` on every OS |
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
2. Zip the source with `make-zip.mjs` (same script as Step 5), excluding build artefacts:
   ```bash
   node scripts/make-zip.mjs . source.zip \
     --exclude=node_modules,.next,dist,out,.git,.turbo,.cache
   ```
   Do not use `zip -r ../source.zip . -x ...` — it has the same Windows-separator hazard as Step 5.
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

---

If a `docs/butterbase/00-state.md` exists in the working directory, prefer invoking via `/butterbase-skills:journey-frontend` so the journey orchestrator stays in sync.
