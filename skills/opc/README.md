# OPC: One Person Company operator

A Claude Code / Codex skill that lets one founder stand up and run a governed, semi to fully
autonomous company over the [Butterbase](https://butterbase.ai) substrate, without designing the
governance themselves.

It is a setup wizard plus an operating harness. The skill configures and orchestrates. The
substrate enforces and remembers. The guardrails (ceilings, approval gates, the action ledger,
the floor) live in the substrate, so the skill can be edited or replaced without weakening safety.

See [`SKILL.md`](./SKILL.md) for the full operating model and [`reference/`](./reference/) for the
onboarding script, governance model, operating loop, and architecture.

## Requirements

OPC's only hard dependency is the **Butterbase MCP** (vMCP), connected with a **substrate-scoped
key**. It does not depend on any other skill.

- A `bb_sk_` key generated with `substrate_access: true`, or a `bb_sub_` key. A plain app key
  returns 403 on substrate routes.
- The substrate must be provisioned for the account (it is lazily created on first use).

## Install

### Option 1: the Butterbase plugin (recommended)

OPC ships in the `butterbase-skills` plugin, which also auto-configures the MCP connection.

```bash
claude plugin add @butterbase/skills
export BUTTERBASE_API_KEY=bb_sk_your_substrate_scoped_key
```

Then in Claude Code, run `/butterbase-skills:opc` or just say "set up my company" / "run today's
ops". The plugin wires the Butterbase MCP automatically.

### Option 2: drop in just this skill

If you already have the Butterbase MCP configured in your environment, copy this folder into your
skills directory:

```bash
cp -r skills/opc ~/.claude/skills/opc          # personal (all projects)
# or
cp -r skills/opc <your-project>/.claude/skills/opc   # project-scoped
```

The `reference/` folder must come along with `SKILL.md`. Make sure your `.mcp.json` (or
environment) has the Butterbase MCP server configured with a substrate-scoped key.

## Two phases

1. **Onboarding (one time, interactive).** About five questions infer your company archetype,
   propose the functions and loops you can actually run, set conservative policy defaults (refund
   ceiling, comms authority, the never-delegatable floor, the escalation channel), and write the
   confirmed governance config into the substrate.
2. **Operating (recurring, autonomous within bounds).** Discrete scoped runs re-ground in the live
   config and current state, propose actions through the policy layer, act within ceilings, stop
   and escalate at boundaries, and produce a briefing.

The autonomy dial moves up over time (hands-on toward YOLO) as you trust the track record. It
never moves the structural floor, and raising it is itself gated by the substrate.
