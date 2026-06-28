---
name: butterbase-skills
description: Claude Code plugin for Butterbase — 30+ guided skills and auto-configured MCP for the AI-native backend-as-a-service.
license: MIT
supported_assistants:
  - claude-code
  - claude-desktop
homepage: https://butterbase.ai
repository: https://github.com/butterbase-ai/butterbase-skills
---

# Butterbase Skills

Claude Code plugin for **[Butterbase](https://butterbase.ai)** — an AI-native backend-as-a-service with Postgres, auth, storage, serverless functions, an AI gateway, RAG, realtime, and durable objects.

This plugin gives Claude deep knowledge of Butterbase's 40+ MCP tools, ships 30+ guided skills, and auto-configures the MCP server connection.

## What you get

- **Auto-configured MCP server** — `.mcp.json` points Claude at `https://api.butterbase.ai/mcp` with your `BUTTERBASE_API_KEY`. All Butterbase tools available immediately.
- **Always-on context** — `CLAUDE.md` teaches Claude Butterbase's environment, branding, tool shape, and core workflow.
- **Guided journey** — `/butterbase-skills:journey` walks any idea from brainstorm → plan → schema → auth → functions → deploy → optional hackathon submission.
- **Per-capability skills** — `schema-design`, `auth-setup`, `function-dev`, `deploy-frontend`, `debug-rls`, `storage`, `rag-dev`, `realtime`, `durable-objects`, `ai`, `migrations`, `substrate`, `integrations`, `payments`, and more.

## Install

```bash
# Add the marketplace
claude plugin marketplace add https://github.com/butterbase-ai/butterbase-skills

# Install the plugin
claude plugin install butterbase
```

## Setup

Sign-in is OAuth — no API key copy-paste needed.

1. **Sign up** at [butterbase.ai](https://butterbase.ai).
2. **Install the MCP server across every detected client:**
   ```bash
   npx @butterbase/cli mcp install
   ```
   The cli walks Claude Code, Cursor, VS Code, JetBrains, Codex, Gemini CLI, and the rest, and prints per-client OAuth hints.
3. **Trigger OAuth once per client.** In Claude Code: restart, then run `/mcp` (or `claude mcp login butterbase`).

## Headline skills

| Skill | What it does |
|-------|--------------|
| `/butterbase-skills:journey` | End-to-end orchestrator — idea → deployed app |
| `/butterbase-skills:build-app` | Build a complete app from scratch |
| `/butterbase-skills:schema-design` | Design Postgres schemas with the declarative DSL |
| `/butterbase-skills:auth-setup` | Configure OAuth providers, JWT, service keys |
| `/butterbase-skills:function-dev` | Develop and deploy serverless functions |
| `/butterbase-skills:deploy-frontend` | Deploy React / Next.js / static to a live URL |
| `/butterbase-skills:debug-rls` | Debug Row-Level Security access issues |
| `/butterbase-skills:rag-dev` | Build a RAG knowledge base with semantic search |
| `/butterbase-skills:durable-objects` | Stateful per-key actors (chat rooms, multiplayer, rate limiters) |
| `/butterbase-skills:realtime` | WebSocket subscriptions for live database changes |
| `/butterbase-skills:integrations` | Composio toolkits — email, Slack, GitHub, Notion, Linear, CRM |
| `/butterbase-skills:payments` | Stripe Connect via `manage_billing` — subscriptions, marketplace splits |
| `/butterbase-skills:substrate` | Per-user agent memory backend (entities, decisions, action ledger) |

See [`CLAUDE.md`](./CLAUDE.md) and the [`skills/`](./skills/) directory for the full set.

## Supported AI assistants

- **Claude Code** (primary) — installed via the plugin marketplace command above; `npx @butterbase/cli mcp install` also covers it.
- **Claude Desktop / Cursor / VS Code / JetBrains / Codex / Gemini CLI / others** — `npx @butterbase/cli mcp install` writes the right config for each, then a single `/mcp` (or the client's equivalent) launches the OAuth flow in your browser.
- Any other MCP-capable client can connect to `https://api.butterbase.ai/mcp` directly — the server advertises OAuth 2.1 via the standard `.well-known/oauth-protected-resource` metadata. Or run the MCP server locally via [`@butterbase/mcp`](https://www.npmjs.com/package/@butterbase/mcp).

## Related

- **[@butterbase/mcp](https://www.npmjs.com/package/@butterbase/mcp)** — stdio MCP server for non-Claude-Code clients
- **[@butterbase/sdk](https://www.npmjs.com/package/@butterbase/sdk)** — TypeScript SDK for app code
- **[@butterbase/cli](https://www.npmjs.com/package/@butterbase/cli)** — CLI for local dev and project scaffolding
- **[butterbase-oss](https://github.com/butterbase-ai/butterbase-oss)** — open-source backend runtime

## License

MIT
