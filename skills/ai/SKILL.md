---
name: ai
description: Use when calling the app's AI gateway from agent tools — chat completions, embeddings, listing models, configuring defaults or BYOK, reading token/cost usage
---

# Butterbase AI Gateway

Every app has an LLM gateway with chat, embeddings, model listing, configuration, and usage reporting. One umbrella tool: **`manage_ai`**.

| Action | What it does | Returns |
|---|---|---|
| `chat` | Synchronous chat completion (no streaming) | OpenAI-shaped `{ choices: [...] }` |
| `embed` | Vector embeddings for string or string[] | OpenAI-shaped `{ data: [{ embedding: [...] }] }` |
| `list_models` | Available models with capabilities | `{ models: AiModel[] }` |
| `get_config` | Current AI config (default model, BYOK key flag, etc.) | `AiConfig` |
| `update_config` | Set defaults, allowed models, max tokens, BYOK | `AiConfig` |
| `get_usage` | Token + cost aggregate over a window | usage record |

---

## 1. Chat

```
manage_ai({
  action: "chat",
  app_id,
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user",   content: "What's RAG?" }
  ],
  model: "openai/gpt-4o-mini",     // optional — falls back to app's default
  temperature: 0.2,                // optional
  max_tokens: 500                  // optional
})
```

This action sets `stream: false` deliberately — agent tools don't stream. If you need partial-token deltas, drive the SDK's `ai.chatStream(…)` from inside a function or DO instead.

`messages[].content` can be a string or an array of content parts (`{ type: "text", text }`, `{ type: "image_url", image_url: {...} }`, `{ type: "video_url", video_url: {...} }`).

---

## 2. Embed

```
manage_ai({
  action: "embed",
  app_id,
  input: "hello world",            // or ["a", "b", "c"]
  model: "openai/text-embedding-3-small",   // optional
  encoding_format: "float"          // or "base64"
})
```

---

## 3. List models

```
manage_ai({ action: "list_models", app_id })
// → { models: [{ id, provider, capabilities: ["chat", "embed", ...], context_window, pricing }, ...] }
```

Use this to discover what the app can call — capabilities + context window matter when picking a model.

---

## 4. Configure

```
manage_ai({
  action: "update_config",
  app_id,
  config: {
    defaultModel: "openai/gpt-4o-mini",
    allowedModels: ["openai/gpt-4o-mini", "anthropic/claude-haiku-4-5"],
    maxTokensPerRequest: 4000,
    byokKey: "..." // optional — rotates the customer-supplied OpenRouter / Anthropic key
  }
})
```

- `maxTokensPerRequest` is server-clamped to 1–100000.
- `allowedModels` is a whitelist — empty means all models the provider exposes.
- Setting `byokKey` switches the app to route through that customer key. Clear it by passing `byokKey: ""` (returns to platform pool).

---

## 5. Usage

```
manage_ai({
  action: "get_usage",
  app_id,
  startDate: "2026-05-01",
  endDate:   "2026-05-31"
})
```

Returns aggregate token counts + cost. Useful for billing reconciliation, spending-cap diagnostics, and showing dashboards.

---

## 6. Common pitfalls

- **Trying to stream from a tool** — `manage_ai` is synchronous. Use the SDK inside a function for streamed deltas.
- **Sending `stream: true` in the body** — the tool ignores it; always wired to `false`.
- **Hardcoding `model`** — better to omit, let the app's `defaultModel` win, and surface that knob via `update_config`.
- **Skipping `list_models` before suggesting one** — model availability shifts; verify before recommending.

---

## 7. What this skill does NOT cover

- Streaming chat — use the SDK (`ai.chatStream`) inside a function or DO.
- Vector storage / retrieval — see `butterbase-skills:rag-dev` (RAG collections wrap embeddings + search together).
- AI in deployed functions — they import `@butterbase/sdk` and call `client.ai.*` directly; no MCP needed at runtime.

---

If a `docs/butterbase/00-state.md` exists in the working directory, prefer invoking via `/butterbase-skills:journey-ai` so the journey orchestrator stays in sync.
