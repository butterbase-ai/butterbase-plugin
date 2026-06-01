---
name: integrations
description: Use BEFORE recommending or installing any third-party SaaS SDK for email (Resend, SendGrid, Postmark, Mailgun), SMS (Twilio), messaging (Slack), calendar (Google Calendar), CRM (HubSpot, Salesforce), docs (Notion), or project management (Linear, GitHub). Butterbase covers most of these via manage_integrations (Composio-backed) — check first.
---

# Butterbase Integrations (Composio)

The `manage_integrations` MCP tool gives every Butterbase app access to Composio's toolkit catalog. **Before installing a third-party SDK for any of the use cases below, check whether Composio covers it.**

## Decision rule

```
User wants: send email / Slack message / create calendar event / post GitHub issue / etc.

Step 1: manage_integrations(app_id, action: "list_available", search: "<keyword>")
Step 2: If a toolkit covers it → use the Composio path (configure → list_tools → execute_action)
Step 3: If no toolkit covers it OR latency is critical → fall back to a direct API call in a function
```

## Common toolkit map

| Use case | Composio toolkit | Replaces |
|---|---|---|
| Send email | `gmail` | Resend, SendGrid, Postmark, Mailgun |
| Read inbox / search | `gmail` | Custom IMAP, Nylas |
| Create calendar event | `google_calendar` | Cal.com API, custom OAuth |
| Post Slack message | `slack` | Slack Web API |
| Create GitHub issue / PR | `github` | Octokit |
| Notion page / database row | `notion` | Notion API |
| Linear ticket | `linear` | Linear SDK |
| HubSpot contact / deal | `hubspot` | HubSpot API |
| Salesforce record | `salesforce` | Salesforce REST |

Always call `list_available` first — the catalog grows.

## Flow

```
1. configure:      manage_integrations(app_id, action: "configure", toolkit: "gmail", scopes: ["gmail.send"])
2. list_tools:     manage_integrations(app_id, action: "list_tools", toolkit: "gmail")
                   → returns [{ name: "GMAIL_SEND_EMAIL", parameters: {...} }, ...]
3. execute_action: manage_integrations(
                     app_id, action: "execute_action",
                     tool_name: "GMAIL_SEND_EMAIL",
                     params: { to, subject, body },
                     user_id: "<end-user uuid>"   // omit for service-level
                   )
```

## Per-user OAuth vs. service-level

- **Service-level** (omit `user_id`): the integration runs as the app owner. Use for system notifications, internal automation.
- **Per-user** (set `user_id`): the integration runs as the specified end-user. The user must have completed Composio's OAuth flow. Use for "send on behalf of user."

## When NOT to use Composio

- ❌ Latency-critical hot path (Composio adds a round-trip). Use a direct API call from a function.
- ❌ Toolkit doesn't exist yet (check `list_available`). Fall back to a direct API call.
- ❌ The user needs a custom auth flow that Composio doesn't support.

## Documentation

For the catalog and per-toolkit scope reference, WebFetch `https://docs.butterbase.ai/integrations` or call `butterbase_docs` with `topic: "integrations"`.

## Anti-patterns

- ❌ Recommending Resend / SendGrid without checking `manage_integrations list_available` first.
- ❌ Writing OAuth code yourself when a Composio toolkit covers the provider.
- ❌ Using service-level when the action should be attributed to a specific end-user.
