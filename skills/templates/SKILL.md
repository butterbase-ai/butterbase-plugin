---
name: templates
description: Use when publishing a Butterbase app as a public template (visibility + listed + repo snapshot), browsing the template gallery, or cloning a template with environment-variable preflight. Templates are public apps with a pushed repo snapshot that other users can fork into their own region.
---

# Butterbase Templates

A "template" is a Butterbase app marked `visibility: 'public'` and `listed: true`, **with a repo snapshot uploaded via `butterbase repo push`**. The clone pipeline replays the source app's schema, RLS, functions, config, and frontend artifact into a new app in the cloner's region; the pushed repo gives the cloner the actual source tree to keep developing.

## How the repo snapshot flows (`repo push` / `repo pull` / `repo status`)

The repo snapshot is the **source-code half** of a template; the clone job replays schema/RLS/functions/config separately. Both halves are pinned via `apps.repo_latest_snapshot`.

- **`butterbase repo init <app_id>`** — Bind the current directory to an app. Writes `.butterbase/config.json` and seeds `.butterbaseignore`. Run once per project. `butterbase clone` does this for you on the cloner side.
- **`butterbase repo push -m "<msg>"`** — Publisher uploads. Walks the project tree (respecting `.butterbaseignore` + `.gitignore`), hashes files (SHA256), uploads any new blobs via presigned URLs, commits a new immutable snapshot, updates the local pin. Content-addressed — re-pushing an unchanged tree is a no-op upload. **This is the only way the source tree gets onto the snapshot.** Clone replay does not synthesise it.
- **`butterbase repo pull [--force]`** — Cloner / collaborator downloads. Fetches the remote latest manifest, diffs against the locally pinned snapshot, downloads changed/new files, deletes files that are gone upstream, and bumps the local pin. Refuses if a file was deleted upstream but modified locally — `--force` overrides. `butterbase clone` invokes this **automatically** after the clone job completes, so cloners get a working tree without thinking about it.
- **`butterbase repo status`** — Inspector. Shows files modified vs pinned, untracked locally, and deleted locally. Use before `push` to confirm what's about to be uploaded; use after `pull` to confirm a clean tree.
- **`butterbase repo log`** — List all snapshots for the bound app.
- **`butterbase repo wipe`** — Destructive; clear all snapshots. Requires app-id confirmation.

**Reading a template's source code:** clone it into a scratch directory — `butterbase clone <source_app_id> /tmp/peek` — and read the resulting tree. There is no separate "browse template source" CLI; snapshots are only materialised by `repo pull`, which requires the caller to be bound to the app.

**Updating a clone with newer template versions:** clones don't auto-follow the source — each clone owns its own app and its own snapshot history. `butterbase repo pull` in the clone fetches the cloner's *own* latest, not the upstream publisher's. To track upstream changes, the cloner has to clone again into a sibling directory and merge with git, or the publisher has to push to a snapshot the clone can read (uncommon).

## What clone replay DOES and DOES NOT copy

| Copied on clone | Not copied — cloner must do |
|---|---|
| Schema (DDL) | App user data (unless explicitly seeded) |
| RLS policies | Auth provider OAuth secrets |
| Functions (handlers + non-secret env) | Function secrets / API keys (cloner provides via preflight) |
| App config | **Agents** (record in `agents` table) — bundle as `agents/*.json` in the repo and re-import after clone |
| Frontend artifact (with app-id rewritten) | **Storage objects** (only the bucket config replays) |
| Repo snapshot (files) | OAuth integration installs |

This makes the **README in the pushed repo** the most important publishing artifact — it has to explain everything that doesn't replay.

## When to use

- Publishing an app you built as a public template.
- Listing or searching templates.
- Cloning a template (or guiding a user through clone with env-var preflight).
- Toggling visibility / listed status on an existing app.

## Publishing checklist

Before flipping `visibility: public`, verify all of the following. Each one prevents a class of broken-on-clone bugs.

1. **Repo snapshot exists and is fresh.** Run `butterbase repo push -m "publish v1"` from the project root. Confirm `apps.repo_latest_snapshot` is non-null via `manage_app action: get`. The clone preflight endpoint returns `has_repo: false` if missing, and discovery hides templates without a repo for several sort modes.

2. **README.md at repo root with clone instructions.** No formal "template instructions" field exists — the README in the pushed snapshot **is** the instructions. It MUST cover:
   - **One-line pitch** (what does this app do?)
   - **Required env vars per function** — the preflight surfaces keys but not what they're for. List each (e.g., `STRIPE_SECRET_KEY — server-side Stripe secret`).
   - **Auth setup** — OAuth provider IDs/secrets the cloner must register (OAuth secrets do not replay).
   - **Agent re-import** — if the app uses agents, the spec files (`agents/*.json`) ride along in the repo, but the `agents` DB rows don't. Document: `butterbase agents create -f agents/<name>.json`.
   - **MCP server registration** — if agents reference MCP servers, document each server's URL/transport/auth-header so the cloner can register them post-clone.
   - **Seed data steps** — if the app needs minimum rows (e.g., a default category), provide a seed script or document `manage_schema` / `select_rows` calls.
   - **First-run smoke** — concrete command(s) to verify the clone works.

3. **Env-var conventions for auto-mint.** If a function reads `butterbase_api_key`, the clone preflight auto-mints a scoped key for the cloner. Use this convention name exactly so the clone modal can fill it without asking.

4. **No secrets in committed files.** `.butterbaseignore` covers the obvious cases (`.env`, `secrets/`). Verify with `butterbase repo push --dry-run` and scan the file list.

5. **Visibility flip.** Only after 1–4: `butterbase visibility public --listed` (or `manage_app action: update_visibility` with `{visibility: 'public', listed: true}`).

## Procedure: publishing

1. **Confirm intent.** `"Publish '<app_name>' (id=<app_id>) as a public, listed template? This makes the app discoverable, exposes the repo snapshot, and lets anyone fork it. (yes / unlisted-public / no)"`.

2. **Author README.** If `README.md` does not exist or is shorter than ~30 lines, write one using the checklist above. Show it to the user for approval before pushing.

3. **Bundle agent specs** if applicable. Confirm `agents/*.json` files exist and are tracked. If the app has agents but no spec files, export each via `butterbase agents get <name>` (the wrapped agents skill helps) and write them to `agents/`.

4. **Dry-run the push.** `butterbase repo push --dry-run`. Review the file list with the user — especially flag any path that looks like a secret.

5. **Push.** `butterbase repo push -m "publish v1"`. Confirm the snapshot ID printed.

6. **Flip visibility.** `butterbase visibility public --listed` (or `unlisted` if step 1 chose "unlisted-public"; unlisted public apps are still cloneable by anyone with the app_id but don't show in discovery).

7. **Verify discovery.** Hit `GET /v1/templates?q=<app_name>` (or `butterbase templates --q <name>`) and confirm the app shows with the right `schema_summary` (table_count, function_count) and `has_repo: true`.

8. **Clone-test it yourself.** `butterbase clone <app_id> /tmp/clone-test-$(date +%s)` and walk the README. Anything ambiguous or missing in the README → fix in the source repo, `butterbase repo push` again (snapshots are content-addressed, so unchanged files don't re-upload).

## Procedure: cloning

1. **Preflight.** `GET /v1/templates/<source_app_id>/clone-preflight` (or the `clone_app` MCP tool with `dry_run: true` if available). Returns per-function env var keys grouped by convention (`auto_mint_eligible` for `butterbase_api_key`).

2. **Collect env vars.** For each non-auto-mintable key, ask the user for the value. For auto-mintable keys, default to "yes, mint a fresh `bb_sk_*`".

3. **Start clone.** `clone_app` (MCP) or `butterbase clone <source_app_id>` (CLI). Pass `env_var_values` and `auto_mint_api_key`.

4. **Poll the job.** Stage progression: `pending → processing → replaying_schema → replaying_rls → seeding_data → replaying_functions → replaying_config → copying_repo → completed`. If it transitions to `failed`, surface the `error_message`; the user can `retry_clone(job_id)`.

5. **Post-clone.** Read the cloned app's README.md (it came down with the repo snapshot). Walk the post-clone steps: OAuth registration, agent re-import (`butterbase agents create -f agents/*.json`), MCP server registration, seed data.

6. **First-run smoke.** Run the README's "first-run smoke" commands.

## CLI reference

| Command | Purpose |
|---|---|
| `butterbase repo init <app_id>` | Bind the current directory to an app. Seeds `.butterbase/config.json` + `.butterbaseignore`. |
| `butterbase repo push [-m msg] [--dry-run]` | Walk files, hash, upload missing blobs, commit snapshot, update `apps.repo_latest_snapshot`. |
| `butterbase repo pull [--force]` | Download the bound app's latest snapshot, diff vs local pin, apply downloads + deletes, bump pin. Refuses on local-edit vs upstream-delete conflict without `--force`. Auto-invoked by `butterbase clone`. |
| `butterbase repo status` | Show files modified vs pinned, untracked, and locally deleted. Use before `push` and after `pull`. |
| `butterbase repo log` | List snapshots for the bound app. |
| `butterbase repo wipe` | Destructive — clear all snapshots. Requires app-id confirmation. |
| `butterbase visibility public [--listed | --unlisted]` | Mark public + optionally list in gallery. |
| `butterbase visibility private` | Unpublish. Auto-unlists. |
| `butterbase templates [--q <text>] [--sort recent\|popular]` | Browse the gallery. |
| `butterbase clone <source_app_id> [target_dir]` | Clone a template into the user's account, replaying source and pulling the repo to `target_dir`. |

## Anti-patterns

- ❌ Flipping `visibility: public` before `butterbase repo push`. Cloners get an empty file tree.
- ❌ Skipping the README. The clone preflight tells the cloner *which* env vars to set, not *what they're for*; without a README, they're guessing.
- ❌ Assuming agents come with the clone. They don't — bundle `agents/*.json` and document re-import.
- ❌ Storing OAuth secrets or function API keys in repo files. They'll be uploaded into every clone's filesystem. Use `.butterbaseignore` and document the env-var keys instead.
- ❌ Publishing without a self-clone test. Always `butterbase clone` your own template into a scratch dir and walk the README before announcing.
- ❌ Confusing `--unlisted` public with `private`. Unlisted-public apps are still forkable by anyone who knows the `app_id` (useful for "share a link with a friend" but not for personal apps).
