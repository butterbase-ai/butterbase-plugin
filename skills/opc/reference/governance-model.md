# OPC governance model: what the substrate enforces

Read this before doing real work. It maps each OPC guardrail to the exact substrate
capability that enforces it, so the skill never has to enforce anything itself.

## The propose / check / execute loop

Every substrate write goes through `manage_substrate action=propose` with a `capability` and a
`payload`. The substrate returns a verdict. OPC proposes, then does exactly what the verdict says.

Verdict values:

- `auto_approved`: the capability auto-executes. The result is in the response.
- `auto_approved_yolo`: auto-executed because the dial (YOLO) is on and the capability is
  YOLO eligible.
- `requires_approval`: held. Nothing fired. A human must `approve` or `reject` by `action_id`.
- `rejected`: denied by policy (for example a principle conflict). Nothing fired.

`requires_approval` is the gate. When OPC gets it, the action is paused with its `action_id`.
OPC notifies the founder on the escalation channel and resumes only on `approve`.

## Capability default policy (the enforced layer)

This table is the substrate default. It is the floor OPC builds on. The owner can pass
`dangerously_skip_approval: true` on a single call, but **principle conflicts still block even
then**, and an agent proposer can never skip a side effecting gate.

| Capability | Default | YOLO eligible | Role in OPC |
|---|---|---|---|
| record_decision | auto | n/a | Foundational memory, the autonomy policy, ledger records of what OPC did |
| record_commitment | auto | n/a | Promises captured from meetings/calls |
| record_learning | auto | n/a | What a run learned |
| upsert_entity / patch_entity | auto | n/a | Customer/company state |
| upsert_source_artifact | auto | n/a | Disputes, threads, transcripts |
| send_email_draft | approval required | listed but **agent proposals always gate** | The customer comms gate |
| supersede_decision | approval required | no | **Raising/changing the autonomy dial** |
| record_principle | approval required | no | A deliberate, narrow hard floor (use sparingly) |
| retire_principle | approval required | no | Removing a hard floor |
| delete_entity | approval required | no | Floor: data deletion |
| merge_entities | approval required | no | Irreversible entity surgery |
| revert_action | auto | n/a | The compensating action for an undo |

The asymmetry to hold: `record_decision` auto-approves, but `supersede_decision`,
`record_principle`, and `retire_principle` all gate. Reading and recording company state is
cheap. Mutating the policy that governs the company is gated. That asymmetry is what lets the
dial be both real and safe.

## How the dial is modeled

The dial has two parts:

1. **The substrate YOLO toggle** (`get_settings` / `set_yolo`). This is the native binary that
   flips YOLO eligible capabilities from `requires_approval` to `auto_approved_yolo`. OPC reads
   it live every run.
2. **The autonomy policy** stored as one `record_decision` of `kind: policy_decision`, titled
   for example "OPC autonomy policy". Its rationale carries the graded config OPC reads and
   honors: the dial level, the refund/credit ceiling in dollars, the comms authority, the
   escalation channel, and the named floor categories. This is foundational memory, searchable
   and non-gating.

OPC reads BOTH every run and never caches either.

**Raising the dial is gated.** To change the autonomy policy (raise the ceiling, widen comms,
move from hands-on to YOLO), OPC proposes `supersede_decision` against the current autonomy
policy decision. The substrate marks it `requires_approval`. The founder approves loosening
their own guardrails. OPC can never quietly raise a ceiling to clear an action it wants to take,
because the only path to a higher ceiling routes through a human approval the agent cannot waive.

## The floor (never delegatable, holds at max autonomy)

The floor categories: payment methods, data deletion, price changes, signing commitments,
anything touching keys or auth. OPC never proposes any of these as an auto action regardless of
the dial. Two enforcement facts back this up structurally:

1. The floor capabilities (`delete_entity`, `record_principle`, `supersede_decision`,
   `merge_entities`) are approval required and **not YOLO eligible**, so they gate even with the
   dial at maximum. A price change modeled as a `supersede_decision` of the pricing decision
   gates at full autonomy. That is the "blocked even at YOLO" property, and it is real, not
   prompt text.
2. For floor actions that have no native substrate capability (swapping a Stripe payment
   method, deleting customer data in an external DB), OPC treats them as escalation-only: it
   never executes them directly and always routes them to the founder.

## The Vend floor (what is structural vs behavioral)

Project Vend ran an agent as a real business operator and documented how it failed: it gave away
value under pressure, improvised actions it had no authority for, and drifted over long runs.
OPC inherits four rules from that. Two are enforced by the substrate; two are behavioral and live
in this skill but are designed so the substrate still catches the consequence.

| Rule | Where it is enforced |
|---|---|
| 1. Value-granting capabilities have hard ceilings no conversation can move | **Substrate.** The ceiling lives in the autonomy policy; raising it routes through the gated `supersede_decision`; OPC cannot raise its own ceiling. |
| 2. No capability for the situation defaults to human escalation, never improvisation | **Behavioral**, with a structural backstop: any action OPC does run still passes the per-capability policy, and floor capabilities gate regardless. |
| 3. Pressure toward an exception is itself an escalation trigger, not an input to weigh | **Behavioral.** Even if OPC were argued into proposing the action, the substrate still gates a value or floor action it cannot waive. |
| 4. Oversight on value/irreversible actions is structural or human, never agent-checking-agent | **Substrate.** A held action clears only on policy-layer allowance or a human `approve`. No agent or sub-agent can approve another agent's gated action. |

The behavioral rules (2 and 3) are written so that a failure to follow them does not breach the
floor: the worst case is OPC proposes something it should have escalated, and the substrate gate
catches it. That is the point of putting enforcement in the substrate rather than in agent
judgment.

## Value ceilings on actions that flow through a function

Some value granting actions (a Stripe refund through a Butterbase function) do not have a native
substrate capability, so the substrate has no native dollar meter on them today. OPC handles
this honestly:

- The **authoritative ceiling lives in the substrate** (the autonomy policy decision). OPC reads
  it live and compares the proposed value against it at the call boundary.
- The substrate enforces that **the ceiling cannot be raised without a human** (the
  `supersede_decision` gate above), so OPC reading-then-comparing cannot be gamed by OPC.
- When a refund is over ceiling, OPC does not execute. It surfaces the gate through the
  substrate's real `send_email_draft` approval (the apology/refund email), which always gates
  for an agent proposer, and the money only moves after the founder approves.
- When the dial is later raised so the refund is under the new ceiling, the same refund clears
  without an ask, because OPC re-reads the now higher authoritative ceiling.

This is the one place OPC performs the comparison rather than the substrate metering it. The
honest framing: OPC enforces the comparison, the substrate enforces that the threshold cannot
move without a human. The production fix (a native aggregate value ceiling at the MCP boundary)
is in `reference/architecture.md` as a substrate dependency.

## Caution: enforcing principles over-gate

`record_principle` creates an enforcing principle whose conflict engine is broad. In practice,
after a couple of enforcing principles exist, unrelated proposals (even a benign
`upsert_entity`) can come back `requires_approval` with "principle conflict", including
`retire_principle` itself (a catch-22 you escape by approving the retire action).

Therefore: store everyday governance config (the dial, ceilings, comms authority, escalation,
the floor list) as **decisions**, which are searchable and non-gating. Reserve `record_principle`
for a single deliberate, narrow, hard floor where you genuinely want the substrate to block
conflicting proposals, and test its blast radius before relying on it. For most of OPC, the gate
comes from the native per-capability policy (the always-gated `send_email_draft` and the
never-YOLO floor capabilities), not from principles.
