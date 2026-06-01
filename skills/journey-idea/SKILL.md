---
name: journey-idea
description: Use as stage 1 of the Butterbase journey, when the user has only a rough idea ("I want to build something that..."). Conducts a concrete, one-question-at-a-time brainstorm that surfaces who the user is, what they do first, what the must-haves are, and inline-tags Butterbase capabilities (→ manage_schema, → deploy_function, etc.). Produces docs/butterbase/01-idea.md.
---

# Journey: Idea brainstorm

Stage 1 of the guided journey. Turn a vague idea into a written brief with capability tags.

## When to use

- Dispatched by `journey` when `current_stage: idea`.
- Directly via `/butterbase-skills:idea`.

## Inputs

None required. Optionally reads any existing `01-idea.md` to resume.

## Procedure — questioning discipline

Follow the cross-cutting rules from the spec:
- **One question at a time.** Never batch.
- **Multiple-choice preferred**, with 2–4 concrete options.
- **Concrete, not abstract.**
- **Recommend** when a Butterbase primitive is obvious.
- **Tag inline.** When a feature surfaces, append the capability tag immediately (`→ manage_schema`, `→ deploy_function`, `→ manage_storage`, `→ manage_oauth`, `→ manage_rag_content`, `→ manage_realtime`, `→ manage_durable_objects`, `→ manage_ai`).
- **Write as you go.** After each answered question, append the decision to `01-idea.md` so progress survives a crash.

Walk through these questions in order. Do not skip — but adapt the wording.

1. `"In one sentence, what is the painful thing your user does today that this replaces?"`
2. `"Who is the first user? ① solo creator ② small team ③ end customers of a business ④ Other"`
3. `"What is the first action they take in the app?"`
4. `"If we could ship only ONE screen, which one?"`
5. `"Do two different users see different data when they log in? (yes → user-scoped data → manage_rls; no → shared → simpler)"`
6. `"Are there files/images involved? (yes → manage_storage; no → skip)"`
7. `"Does anything happen on a schedule or in response to an external event? (yes → deploy_function cron/webhook; no → skip)"`
8. `"Is there a knowledge base / search-over-documents feature? (yes → manage_rag_content; no → skip)"`
9. `"Does the UI need live updates or presence? (yes → manage_realtime; no → skip)"`
10. `"Any chat rooms / multiplayer / per-user-actor state? (yes → manage_durable_objects; no → skip)"`
11. `"Any LLM features (chat, embeddings, summarisation)? (yes → manage_ai; no → skip)"`

If `hackathon_mode: true` in `00-state.md`: after question 4, also ask:
- `"Hackathon deadline (ISO date/time)?"` — write to `00-state.md` front-matter.
- After each subsequent feature ask `"Can this ship in time, or move to a 'post-hackathon' section?"`

## `01-idea.md` format

```markdown
# Idea

**One-liner:** <user's answer to Q1>

**First user:** <Q2>
**First action:** <Q3>
**One screen:** <Q4>

## Must-haves
- <feature> → <capability tag>
- ...

## Post-hackathon (nice-to-have)
- ...

## Capability map
| Capability | Used? | Why |
|---|---|---|
| manage_schema | yes | core data model |
| manage_rls | yes | user-scoped data |
| manage_oauth | yes | Google sign-in |
| manage_storage | no | no files |
| deploy_function | yes | daily-digest cron |
| manage_ai | no | |
| manage_rag_content | no | |
| manage_realtime | no | |
| manage_durable_objects | no | |
```

### Toolchain note

At the end of the idea write-up, also note in one line that the build will use:
- `@butterbase/sdk` for any frontend / Node app code
- `@butterbase/cli` for the local dev loop (scaffolding, logs, key generation)
- The MCP tools (this plugin) for orchestrating provisioning, deployments, integrations

This sets expectations before `plan` makes specific package choices.

## Outputs

- Writes `docs/butterbase/01-idea.md`.
- Ticks `- [x] idea` and bumps `current_stage: plan`, `last_updated` in `00-state.md`.

## Anti-patterns

- ❌ "Tell me more about your app." Use the concrete questions above.
- ❌ Asking 3 questions at once.
- ❌ Accepting "all of the above" in hackathon mode — force prioritisation.
- ❌ Moving on before writing `01-idea.md` to disk.
