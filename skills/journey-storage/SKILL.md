---
name: journey-storage
description: Use as the storage build stage of the Butterbase journey. Implements the Storage section of 02-plan.md by delegating to the storage skill. Calls manage_storage (update_config) for bucket setup; uploads are exercised at frontend integration time, not here. Skipped if the plan has no file uploads.
---

# Journey: Storage

Stage 3d of the guided journey. Configure storage buckets/ACLs from the plan.

## When to use

- Dispatched by `journey` when `current_stage: storage`.
- Directly via `/butterbase-skills:journey-storage`.
- Skipped (annotated `(n/a)`) if the plan has no Storage section.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the Storage section.
- `docs/butterbase/00-state.md` — for `app_id`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "storage"`. For presigned URL patterns, also WebFetch `https://docs.butterbase.ai/storage`. Skip if cache is fresh.

1. Read the Storage section of `02-plan.md`. Print it back: `"About to configure storage: <buckets>. Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:storage` via the Skill tool, passing the Storage plan and `app_id`. The wrapped skill calls `manage_storage action: update_config` per bucket (visibility, max object size).
3. Smoke: request an upload URL via `manage_storage action: upload_url` and a matching `download_url` to confirm presigned URL minting works.
4. Append one line to `docs/butterbase/04-build-log.md`:
   `<ISO timestamp>  storage  manage_storage  ok`
5. Tick `- [x] storage` in `00-state.md`, set `current_stage:` to the next unchecked stage.
6. Return to `journey` orchestrator (or ask `"Continue to the next stage? (yes/no)"`).

## Outputs

- Configured storage buckets.
- One line in `04-build-log.md`.

## Anti-patterns

- ❌ Persisting `s3_key` in app data — always persist `object_id`. The wrapped skill handles this; do not override it.
