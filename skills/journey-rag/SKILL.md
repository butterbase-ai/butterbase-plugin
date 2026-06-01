---
name: journey-rag
description: Use as the RAG build stage of the Butterbase journey. Implements the RAG section of 02-plan.md by delegating to rag-dev. Calls manage_rag_content (create_collection, ingest_document). Skipped if the plan has no knowledge-base feature.
---

# Journey: RAG

Stage 3g of the guided journey. Create RAG collections and ingest initial documents.

## When to use

- Dispatched by `journey` when `current_stage: rag`.
- Directly via `/butterbase-skills:journey-rag`.
- Skipped (annotated `(n/a)`) if the plan has no RAG section.

## Preflight

If `docs/butterbase/03-preflight.md` is missing, older than 24 hours, or `00-state.md` has `app_id: null`, invoke `butterbase-skills:journey-preflight` first. Wait for it to return successfully before proceeding.

## Inputs

- `docs/butterbase/02-plan.md` — the RAG section.
- `docs/butterbase/00-state.md` — for `app_id`.

## Procedure

0. **Refresh docs.** Call `butterbase_docs` with `topic: "rag"`. For ingestion + retrieval patterns, also WebFetch `https://docs.butterbase.ai/ai/rag`. Skip if cache is fresh.

1. Read the RAG section. Print it back: `"About to set up RAG: collections=<list>. Proceed?"`. Wait for `yes`.
2. Invoke `butterbase-skills:rag-dev` via the Skill tool with the RAG plan and `app_id`. The wrapped skill calls `manage_rag_content action: create_collection`, then `ingest_document` for any seed sources the user provides.
3. Smoke: call `rag_query` with a representative question and show the user the top hit.
4. Append one line to `docs/butterbase/04-build-log.md`:
   `<ISO timestamp>  rag  manage_rag_content  ok`
5. Tick `- [x] rag` in `00-state.md`, set `current_stage:` to the next unchecked stage.
6. Return to `journey` orchestrator (or ask `"Continue to the next stage? (yes/no)"`).

## Outputs

- Created RAG collections, ingested seed docs.
- One line in `04-build-log.md`.

## Anti-patterns

- ❌ Ingesting huge document sets in a single call — chunk and poll `get_document_status`.
- ❌ Skipping the smoke `rag_query` — silent embedding failures are common.
