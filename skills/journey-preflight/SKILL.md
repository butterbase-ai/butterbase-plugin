---
name: journey-preflight
description: Use before any Butterbase platform-touching action (or when journey-preflight is invoked directly) to verify the user has a Butterbase account, the MCP server is connected, BUTTERBASE_API_KEY is set, and an app_id exists for this project. Re-run automatically if docs/butterbase/03-preflight.md is older than 24h or any required value is null.
---

# Butterbase Preflight

Gate that ensures the user can actually talk to the Butterbase platform before any stage tries to. Writes results to `docs/butterbase/03-preflight.md` and stamps `app_id` + `api_base` into `00-state.md`.

## When to use

- Automatically as the first step of any `journey-*` build stage when `03-preflight.md` is missing, older than 24 hours, or `00-state.md`'s `app_id` is null.
- Directly via `/butterbase-skills:journey-preflight` when the user wants to re-verify.

## Procedure

For each check, write a one-line result to `03-preflight.md` (`<check> ✓/✗ <note>`). Ask the user to fix any ✗ before continuing — do not auto-skip failures.

1. **Butterbase account.** Call `mcp__butterbase__list_regions` (cheapest call). Success → ✓. Auth error → ✗ with `"Sign up at https://butterbase.ai and create an API key, then export BUTTERBASE_API_KEY."`. If MCP tool is not visible at all, jump to step 2.

2. **MCP server connected.** Check whether `mcp__butterbase__*` tools are listed in the agent's available tools. Absent → ✗ with `"Install/enable the Butterbase MCP server. See https://butterbase.ai/docs/mcp."`. Present → ✓.

3. **API key set.** Run `echo "${BUTTERBASE_API_KEY:-MISSING}"` via Bash. If `MISSING` or empty → ✗ with `"Export BUTTERBASE_API_KEY (starts with bb_sk_) in your shell or .env."`. Otherwise → ✓ (do not echo the key value to the user).

4. **App provisioned.** Read `00-state.md` front-matter for `app_id`.
   - Non-null and `mcp__butterbase__manage_app` (action `get_config`) succeeds → ✓.
   - Null → ask the user: `"No app yet. Create one now? (yes/no — if no, you can paste an existing app_id)"`. On yes, call `mcp__butterbase__init_app` (ask for app name and region — recommend `us-east-1` as default). Capture `app_id` and `api_base` from the response. Update `00-state.md` front-matter.
   - User-provided app_id → verify with `manage_app get_config`; on success update `00-state.md`.

5. **CLI installed.** Run `butterbase --version`. If "command not found":
   - Read the Toolchain section of `02-plan.md`. If `CLI usage: yes` (default), prompt: "Install `@butterbase/cli` globally now? (yes / skip)". On yes, run `npm install -g @butterbase/cli`. On skip, note in `03-preflight.md` that the CLI is not installed and which workflows will require dashboard fallback.
   - If `CLI usage: no`, skip this check.

## `03-preflight.md` format

```markdown
# Preflight — <ISO date>

- account        ✓
- mcp_connected  ✓
- api_key        ✓
- app_provisioned ✓  app_id=app_abc123 region=us-east-1

Re-run with /butterbase-skills:journey-preflight.
```

## Outputs

- Writes/overwrites `docs/butterbase/03-preflight.md`.
- Updates `00-state.md` front-matter `app_id`, `api_base`, `last_updated`, and ticks `- [x] preflight`.

## Anti-patterns

- ❌ Echoing the API key value back to the user.
- ❌ Skipping a failed check ("I'll just try anyway"). Stop and ask the user to fix it.
- ❌ Calling `init_app` without confirming with the user — apps cost money on paid plans.
