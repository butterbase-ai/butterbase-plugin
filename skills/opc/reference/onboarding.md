# OPC onboarding: the interview, archetypes, and the writes

Phase A turns a few high level inputs into a configured company in the substrate. The founder
answers about five questions and confirms drafts. The reliability bet is **archetype plus
adjustment**, never blank slate generation.

## Before the interview: read current state

Never overwrite blindly. First:

1. `manage_substrate action=get_settings` for the current dial.
2. `manage_substrate action=find_entities type=self` for an existing company record.
3. `manage_substrate action=search_memory q="OPC autonomy policy"` for existing config.

If config exists, switch to revise mode: show the founder what is configured and adjust, rather
than re-onboarding.

## The interview (about five questions)

Ask in plain language. Infer the archetype from the answers; do not ask "pick an archetype".

1. **What does the company do, and what stage is it at?** (Infers archetype and ambition.)
2. **What do you sell, and to whom?** (Infers the function set and the comms surface.)
3. **Refund and credit ceiling:** "I will let the agent issue refunds and credits up to a limit
   without asking you. Anything above comes to you. I suggest $20 to start. Keep it?"
4. **Comms authority:** "The agent can reply to customers and send routine emails. It will not
   email investors or send anything to your full customer list without you. OK?"
5. **Escalation channel:** "When the agent needs you, where should it reach you?" (email, Slack.)

## Archetype defaults

Pick one from the first answers, then adjust. Each archetype is a starting configuration, not a
cage.

### SaaS startup (Groomly is this)

- Functions: customer health / churn, customer support, revenue / billing monitoring, founder
  briefing.
- Loops: daily customer-health scan, daily support triage, daily billing-anomaly scan, weekly
  briefing.
- Default ceiling: $20 refund/credit. Comms: routine customer replies only.

### Services SMB

- Functions: client comms, scheduling, invoicing / billing monitoring, founder briefing.
- Loops: daily client-comms triage, daily schedule check, weekly invoice-status scan, weekly
  briefing.
- Default ceiling: $20. Comms: routine client replies only.

### Ecommerce

- Functions: order / fulfillment monitoring, customer support, refund/returns monitoring,
  founder briefing.
- Loops: daily order-exception scan, daily support triage, daily refund/returns scan, weekly
  briefing.
- Default ceiling: $20 refund/credit. Comms: routine buyer replies only.

**Every archetype also gets a commitment follow-through loop.** Promises made to customers and
partners (often captured from a meeting or call) are first-class company state, and acting on
them is where the institutional memory pays off. This loop watches outstanding commitments and
their due dates (`list_memory kinds=commitments`), and as each comes due it acts within ceilings
or escalates. It is the loop that closes "we promised them X" into "we delivered X." Default
cadence: daily. Propose it for every company.

The conservative floor is identical across archetypes and is not adjustable during onboarding:
payment methods, data deletion, price changes, signing commitments, and anything touching keys
or auth always escalate.

## Discover data surfaces, then filter to the runnable set

Archetype proposes the ideal functions. What OPC can actually reach decides which loops run.

- **Product app on Butterbase:** loops query domain state natively (`select_rows`,
  `invoke_function`). Richest case.
- **Product app not on Butterbase:** loops run on the substrate entity graph plus whatever is
  wired through the integration layer (Stripe, their DB, a support inbox). Detect connections
  with `manage_integrations action=list_connected`.
- **CRM recipe present:** customer and at-risk loops read the same substrate the CRM writes, so
  there is no handoff to build. CRM-backed loops are offered only when the CRM is present.
- **Meeting and call capture (opt-in, never default):** voice and conversation are the richest
  feedstock for institutional memory, so meeting capture strongly upgrades every loop's grounding.
  But it is the founder's deliberate choice, not a default. Recording carries real consent and
  legal weight (two-party-consent states, GDPR, employee comfort). Offer it as a connection the
  company turns on; never enable recording by default. The substrate holds the structured,
  governed memory; the company owns the capture decision.

For every proposed loop, confirm its data source is reachable. A loop whose data you cannot
reach becomes a "connect this to unlock this loop" prompt. Name the gap. Never register a loop
that will silently fail.

## Propose, confirm, then write

Show the founder the runnable functions and loops (each with a one line description and a default
cadence), the policy defaults, and the named gaps. On confirmation, write to the substrate.

### 1. Foundational memory (company basics)

`propose capability=upsert_entity` with `type: self` for the company record (mission, what it
does, stage, what it sells, customer type as `attrs`). Then `propose capability=record_decision`
of `kind: mission` for the mission statement so it is searchable foundational memory.

### 2. The autonomy policy (one policy_decision)

`propose capability=record_decision`:

```json
{
  "capability": "record_decision",
  "payload": {
    "title": "OPC autonomy policy",
    "kind": "policy_decision",
    "salience": "ambient",
    "rationale": "Autonomy dial: hands-on. Refund/credit ceiling: $20. Comms authority: routine customer replies only, signed as the operator identity '<Company> Support' (never a named human, never claiming to be human); no full-list sends, no investor email without approval. Escalation channel: <channel>. Never delegatable floor (holds at every dial level): payment methods, data deletion, price changes, signing commitments, keys/auth. Raising any of these values requires founder approval via supersede_decision."
  }
}
```

This auto-executes (it is a decision). It is the single source of truth OPC re-reads every run.
Changing it later goes through `supersede_decision`, which gates.

### 3. The loops (attention rules)

For each confirmed loop, `manage_substrate action=create_rule` with a cron `trigger_cron` and the
loop's scoped intent. These register the loop in the substrate scheduler. In the cockpit demo
they are shown as registered config; the hosted runner executes them unattended in production
(see `reference/architecture.md`). Example daily customer-health rule:

```json
{
  "name": "daily customer-health scan",
  "trigger_cron": "0 9 * * *",
  "condition_mode": "snapshot_predicate",
  "condition": { ">": [{ "var": "entity_count" }, 0] },
  "action_capability": "record_decision",
  "action_payload_template": {
    "title": "Customer-health scan queued",
    "kind": "operational",
    "rationale": "OPC daily customer-health loop is due."
  }
}
```

Use `create_rule` preview semantics before saving when available. The rule is the registration;
the operating logic lives in `reference/operating.md` and runs under the scoped OPC identity.

### 4. Set the dial

Leave `yolo_mode` off (hands-on) by default. If the founder explicitly chooses a higher starting
point, that is their call, surfaced and confirmed, never assumed. `set_yolo` only on explicit
instruction.

## Output of Phase A

Foundational memory set, the runnable functions chosen, the autonomy policy written, escalation
target recorded, loops registered, gaps named. The founder did this by answering questions and
confirming drafts, not by designing a governance system.
