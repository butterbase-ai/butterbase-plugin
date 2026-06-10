---
name: journey-templates
description: Use as the optional template-publishing stage of the Butterbase journey, after deploy (and substrate, if used). Walks the user through authoring a clone-ready README, bundling agent spec files, dry-running and pushing the repo snapshot, flipping visibility to public + listed, and self-clone-testing. Delegates the mechanics to the `templates` skill. Skipped by default in hackathon mode.
---

# Journey: Publish-as-template (optional)

Turn the deployed app into a public, forkable template. This stage is **only** for apps the user explicitly wants to share — most journeys end at `deploy` (or `submit` in hackathon mode).

## When to use

- Dispatched by `journey` when `current_stage: templates`.
- Directly via `/butterbase-skills:journey-templates`.
- The row is optional in the checklist; skip silently if the user declines.

## Inputs

- A deployed, working app (post-`journey-deploy`).
- `docs/butterbase/02-plan.md` — to know what features the app uses (agents, MCP servers, OAuth providers, function env vars).
- `docs/butterbase/04-build-log.md` — to confirm deploy smoke passed.

## Preflight

- `00-state.md` must have `current_stage: templates` or later AND `deploy` ticked.
- The `@butterbase/cli` must be installed locally (`butterbase --version` works). If not, invoke `journey-preflight` to fix.
- The repo must be bound: `.butterbase/config.json` exists with the right `currentApp`. If not, run `butterbase repo init <app_id>`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "templates"` (and `topic: "repo"` if the cache doesn't already cover snapshot push). Skip if `03b-docs-cache.md` is fresh.

1. **Confirm intent.** Ask:
   > "Publish '<app_name>' as a public template? Public + listed makes it appear in the gallery; public + unlisted is forkable by anyone with the app_id but hidden from search. Or skip. (listed / unlisted / skip)"

   Default: **skip** (especially in hackathon mode unless the user explicitly chose to publish their submission).

2. **On skip:** mark the row `- [x] templates (skipped — not publishing)` in `00-state.md`. Write `docs/butterbase/06-template.md` with one line: `Skipped on <date>.` Return to orchestrator.

3. **On listed / unlisted:** delegate to `butterbase-skills:templates`. The wrapped skill walks the user through:
   a. **Author / review README.md.** This is the most important step — clone replay does not carry OAuth secrets, agent records, MCP server registrations, function secrets, or user data. The README must explain every manual step a cloner will need. The `templates` skill has the full checklist; do not let it write a short README and move on.
   b. **Bundle agent specs.** If the app has agents, ensure `agents/<name>.json` exist (export with `butterbase agents get <name>` if missing) and are tracked.
   c. **Inspect drift.** `butterbase repo status`. Walk the modified / untracked / deleted lists with the user. This is the chance to catch a stray secret file or an unwanted directory before it ends up in the snapshot.
   d. **Dry-run.** `butterbase repo push --dry-run`. Confirm the final upload manifest matches what `repo status` showed.
   e. **Push.** `butterbase repo push -m "publish v1"`. Capture the snapshot id.
   f. **Flip visibility.** `butterbase visibility public --listed` (or `--unlisted`).
   g. **Self-clone test.** `butterbase clone <app_id> /tmp/clone-test-<ts>` into a scratch directory — this also exercises the cloner-side `repo pull` that `butterbase clone` runs internally. The wrapped skill walks the README from a cold cloner's perspective and reports any gap. Loop back to (a) if gaps exist.

4. **Write artifact.** `docs/butterbase/06-template.md`:

   ```markdown
   ---
   published_at: <ISO>
   app_id: <id>
   visibility: public
   listed: <true|false>
   snapshot_id: <id>
   selfclone_test: <pass | gaps fixed in round N>
   ---

   # Template publication

   - Visibility: public (<listed | unlisted>)
   - Latest snapshot: <id>
   - README covers: env vars, OAuth, agent re-import, MCP servers, seed data, first-run smoke
   - Self-clone test directory: /tmp/clone-test-<ts>
   - Gallery URL: https://butterbase.ai/templates/<app_id>  (or wherever discovery lives)
   ```

5. **Update state.** Tick `- [x] templates` in `00-state.md`.

6. Return to orchestrator (or end the journey, since this is the last optional stage).

## Anti-patterns

- ❌ Pushing the repo *after* flipping visibility. Order matters: snapshot first, public second. Otherwise the gallery briefly advertises an empty template.
- ❌ Skipping the README. The clone preflight tells the cloner *which* env vars exist but not *what they're for* or what manual setup remains. A bare-bones README publishes a broken-on-clone experience.
- ❌ Skipping the self-clone test. The author always knows too much; only a cold walk-through catches missing instructions.
- ❌ Forgetting agent specs. The `agents` DB table doesn't replay. Without `agents/*.json` in the repo, cloners can see your agent code paths but cannot recreate the agents themselves.
- ❌ Publishing in hackathon mode by default. Hackathon submissions are usually private; only publish if the user explicitly asks.
- ❌ Pushing files containing secrets. Always dry-run first and scan the file list.
