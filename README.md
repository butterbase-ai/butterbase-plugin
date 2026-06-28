# Butterbase Skills

Claude Code plugin for [Butterbase](https://butterbase.ai) — the AI-Native Backend-as-a-Service.

This plugin gives Claude deep knowledge of Butterbase's 42+ MCP tools, guides you through common workflows, and auto-configures the MCP server connection.

## Installation

```bash
claude plugin add @butterbase/skills
```

## Setup

Sign-in is OAuth — no API key copy-paste needed.

1. **Sign up** at [butterbase.ai](https://butterbase.ai).

2. **Install the MCP server across every detected client** with one command:
   ```bash
   npx @butterbase/cli mcp install
   ```
   This walks Claude Code, Cursor, VS Code, JetBrains, Codex, Gemini CLI, and the rest, and prints per-client OAuth follow-up hints.

3. **Trigger OAuth once per client.** In Claude Code: restart, then run `/mcp` (or `claude mcp login butterbase`). The cli's output covers every other client.

The plugin auto-loads its skills and CLAUDE.md context as soon as Claude Code starts. All 42+ tools available immediately after consent.

## Available Skills

| Skill | Description | Example prompts |
|-------|-------------|-----------------|
| `butterbase:build-app` | Build a complete app from scratch | "Build me a blog with auth and comments" |
| `butterbase:schema-design` | Design database schemas | "Design a schema for an e-commerce app" |
| `butterbase:deploy-frontend` | Deploy frontends to live URLs | "Deploy my React app to production" |
| `butterbase:debug-rls` | Debug Row-Level Security | "Users are seeing each other's data" |
| `butterbase:function-dev` | Develop serverless functions | "Create a cron job to clean up expired sessions" |
| `butterbase:contributing` | Contribute to Butterbase | "How do I add a new MCP tool?" |

## What's Included

- **`.mcp.json`** — Auto-configures the Butterbase MCP server connection (HTTPS endpoint)
- **`CLAUDE.md`** — Always-on context: environment variables, workflows, patterns, documentation references
- **6 skills** — Guided workflows for building, deploying, debugging, and contributing

## Local Development

If you're running the Butterbase monorepo locally, the MCP server URL defaults to `http://localhost:4000/mcp`. Set this in your environment:

```bash
export CONTROL_API_URL=http://localhost:4000
```

## Also Available

- **[@butterbase/sdk](https://www.npmjs.com/package/@butterbase/sdk)** — TypeScript SDK for client-side and server-side use
- **[@butterbase/cli](https://www.npmjs.com/package/@butterbase/cli)** — CLI tool for project scaffolding and backend management
- **`butterbase_docs` MCP tool** — Call with any topic (auth, storage, functions, etc.) for comprehensive reference docs

## License

MIT
