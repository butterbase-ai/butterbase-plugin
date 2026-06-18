# OPC architecture: runtime, deployment cases, and substrate dependencies

This file holds the parts of OPC that are design and narration rather than live cockpit code:
the runtime split, the three deployment cases, and the cross-loop coordination the substrate
still owes OPC.

## Runtime split: cockpit vs hosted runner

"Invoke it in Claude Code and it runs 24/7" elides a real architecture decision. A laptop
session is not a server and should not pretend to be one. Split it:

- **Cockpit:** the founder's Claude Code or Codex. Onboarding, reviewing escalations, reading
  briefings, ad-hoc "go do X", raising the dial. This is where the founder grabs the wheel.
- **Hosted runner:** a Butterbase-hosted process that executes the registered loops unattended,
  on schedule and on events, under a scoped OPC identity. This is the autopilot that runs whether
  or not the founder is watching.

Same skill, same vMCP, same substrate, two runtimes. The runner executes under a scoped identity
so the ledger records "OPC-agent did X" and its authority is exactly the policy layer, no more.

**For a demo:** build the cockpit path live and narrate the runner. Show the loops as registered
attention rules (config that exists), and explain that production runs them unattended with the
identical gates. Do not build the runner for a demo. One clean cockpit loop plus the dial beats
four flaky live loops.

## Three deployment cases (design for all, optimize for the first)

1. **Product app built on Butterbase (optimize for this).** Loops query domain state natively
   through app tools. Richest signal, least wiring.
2. **Product app not on Butterbase.** Loops run on the substrate entity graph plus whatever is
   wired through the integration layer (Stripe, their DB, a support inbox). Any function needing
   data you cannot reach becomes a "connect this to unlock this loop" prompt, never a silently
   broken loop.
3. **CRM recipe not cloned or not on Butterbase.** OPC still operates on the substrate.
   CRM-backed loops are offered only when the CRM is present.

So onboarding discovers data surfaces, it does not only infer functions: the archetype proposes
the ideal function set, the reachable connections filter it to the runnable set, and the gaps get
named.

## How OPC and the CRM recipe work together

Recipes are functional surfaces; OPC is the operator. Same substrate underneath, two drivers: a
human logs into the CRM UI, or OPC drives the same CRM over MCP. OPC employs the CRM, it does not
rebuild it.

- The CRM already syncs both ways with the substrate, so there is no handoff to build. OPC reads
  the same contacts, at-risk signals, and meeting-captured decisions the human sees.
- CRM capabilities (email campaigns; social posting; meeting join that parses notes and writes
  decisions and commitments into the substrate) are called by OPC on a loop.
- Meetings feed OPC's grounding, and voice is the richest feedstock of all. Decisions and
  commitments captured from calls land in the substrate as structured, queryable memory, and OPC
  re-grounds on them every run as primary context. Decide "pause enterprise outreach" on a call
  and OPC's next loop respects it. An agent that has been in the room can operate; one on entity
  and business state alone can only report.
- Meeting capture is the company's opt-in choice, never on by default. Recording carries consent
  and legal weight; OPC offers it as a connection the founder turns on. The defensible position is
  that the substrate holds the governed memory fed by whatever capture the company chooses; the
  company owns the recording decision.
- Same gates regardless of driver: a full-list email or a public post is high stakes whether a
  human clicks send or OPC proposes it.

## Cross-loop coordination (substrate dependency, out of demo scope)

Once one agent orchestrates several loops, two failure modes appear that a single recipe never
has. Encode these as substrate dependencies OPC relies on; they are a slide, not demo code.

1. **Conflicting actions on the same entity.** The churn loop offers a retention discount while
   the billing loop flags the same account for a price bump. Per-action ceilings do not catch
   this. The substrate needs a **conflict check**: before a loop acts on an entity, it checks the
   ledger for pending or recent conflicting actions on that entity and yields or escalates.
2. **Compounding value grants.** Support issues a $20 refund, churn issues a $20 credit, billing
   comps a month, same customer, same week. Each clears its own ceiling; together they pass
   intent. The substrate needs an **aggregate ceiling**: a per-entity-per-period value cap that
   every loop draws against, independent of any single action's ceiling.

This is the same structural-not-prompt safety the whole design rests on, applied across loops
instead of within one. Until the substrate provides it natively, OPC runs one primary loop at a
time in the cockpit and treats multi-loop concurrency as a hosted-runner concern gated by these
two checks.

## Other substrate dependencies OPC wants

- **Native value ceiling at the MCP boundary.** Today the substrate has no native dollar meter on
  value actions that flow through a function (a refund). OPC reads the authoritative ceiling from
  the substrate and enforces the comparison at the call boundary; the substrate enforces that the
  ceiling cannot move without a human. The clean version is a native aggregate value ceiling the
  gate evaluates directly, so the comparison is enforced at the chokepoint rather than by the
  caller. See `reference/governance-model.md`.
- **Narrower principle conflict matching.** Enforcing `record_principle` entries currently
  over-gate unrelated actions. Until that is narrowed, OPC stores governance config as decisions
  and reserves principles for a single deliberate hard floor.
- **First-class sensitivity and visibility designations.** Meeting-level (this call is recorded
  but restricted) and item-level (this decision is founder-only) sensitivity should be a native
  designation on memory items and source artifacts, which OPC reads and honors before any
  outward-facing action or briefing. Today OPC enforces this behaviorally and treats unmarked but
  plausibly sensitive content as restricted; the durable version is a substrate-native designation
  system. Governing this context layer (who may read what, who may act on what) is the opening
  OPC's ledger, policy layer, and access controls are built to answer.

## Out of scope tonight (narrate, do not build)

- The hosted 24/7 runner and scheduler (loops shown as registered config).
- Cross-loop coordination and aggregate ceilings (the two checks above).
- Sub-agent, teammate, and contractor routing. The escalation gate already is the route-to-a-human
  primitive; the human it routes to can be the founder, a teammate, or a one off contractor.
- Multiple concurrent loops.
