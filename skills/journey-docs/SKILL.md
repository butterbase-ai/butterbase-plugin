---
name: journey-docs
description: Use as the docs-priming stage of the Butterbase journey, immediately after journey-preflight and before the first build stage. Reads the plan to discover which capabilities the app uses, calls butterbase_docs once per relevant topic, and caches a summary the build stages can re-read.
---

# Journey Docs Gate

A short, mechanical priming step. The goal is that every downstream stage starts with the relevant docs already in the conversation, so the model doesn't invent API shapes.

## When to use

Invoke automatically after `journey-preflight` returns. Also invoke standalone (`/butterbase-skills:journey-docs`) any time the user changes the plan or you realize a stage is using a capability you haven't refreshed this session.

## Procedure

1. **Read the plan.** Open `docs/butterbase/02-plan.md`. Identify every Butterbase capability in use (Tables → schema; RLS → auth; Auth → auth; Storage → storage; Functions → functions; AI → ai; RAG → rag; Realtime → realtime; Durable → functions; Frontend → frontend; Integrations → integrations; Substrate → substrate; Payments → billing).

2. **Call `butterbase_docs` per topic.** For each unique topic from step 1, call `butterbase_docs` with that `topic` argument. Skip duplicates.

3. **Write the cache file.** Create `docs/butterbase/03b-docs-cache.md` with this structure:

   ```markdown
   ---
   primed_at: <ISO timestamp>
   topics: [<comma-separated list>]
   ---

   # Docs Cache

   ## <topic-1>
   <one-paragraph summary of what the MCP doc says — key endpoints, common patterns, gotchas>
   URL: https://docs.butterbase.ai/<area>

   ## <topic-2>
   ...
   ```

4. **Update `00-state.md`.** Tick the `docs` row.

5. **Return.** One-line summary: `"Primed docs for: schema, auth, storage, functions. Cache at docs/butterbase/03b-docs-cache.md."`

## Anti-patterns

- ❌ Calling `butterbase_docs` with `topic: "all"`. Burns context. Call once per relevant topic instead.
- ❌ Skipping the cache file. Other stages re-read it; in-memory only doesn't survive context compression.
- ❌ Calling for topics not in the plan. We're priming, not exhaustive.
