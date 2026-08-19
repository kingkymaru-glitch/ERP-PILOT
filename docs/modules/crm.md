# CRM Module — Structure & Pipeline Proposal (SKETCH)

> Status: DISCUSSION DRAFT. No code. Requires sign-off from MARU + the CRM domain owner before any implementation.
> This document is the pre-coding discussion artifact per the ERP-PILOT dev protocol.

## 1. Module purpose
Turn leads into revenue through a *process*, not a contact spreadsheet. CRM is the first vertical slice that proves the whole ERP-PILOT pattern: deterministic engine + state machine + agentic harness + human approval + LLM reporting.

## 2. Process / state machine (the pipeline)
Every deal is a state machine. Transitions are *events* that fire deterministic tool actions.

```
        ┌─────────┐  qualify  ┌──────────┐  propose  ┌──────────┐  negotiate ┌──────────┐
        │  LEAD   │ ────────▶ │ QUALIFIED│ ────────▶ │ PROPOSAL │ ─────────▶ │NEGOTIATION│
        └─────────┘          └──────────┘          └──────────┘            └────┬─────┘
             │                     │                     │                      │  ┌─────────┐
        score/route           assign owner           draft send             won │  │  WON   │
             │                     │                     │                      ▼  └─────────┘
             │                     │                     │                   ┌─────────┐
             │                     │                     │        lost ◀─────│ LOST   │
             ▼                     ▼                     ▼                   └─────────┘
        (nurture)             (notify)              (forecast upd)
```

- **LEAD** — captured from landing page / signup / manual entry. Event: `lead.created`.
- **QUALIFIED** — scored + assigned. Event: `lead.qualified` → notify owner, scrape account social.
- **PROPOSAL** — proposal drafted + sent. Event: `proposal.sent` → update forecast, log activity.
- **NEGOTIATION** — active deal. Event: `stage.advanced`.
- **WON / LOST** — terminal. Event: `deal.won`/`deal.lost` → reconcile to Finance (order_id), update pipeline.

Each transition writes to `audit_log` with actor (tool|agent|user), timestamp, payload.

## 3. Data model (keys)
Tables (deterministic store, Supabase):
- `leads` (lead_id, customer_id, source, score, state, owner_id, created_at)
- `contacts` (contact_id, customer_id, name, channel_handles)
- `accounts` (customer_id PK, name, industry, social_handles)
- `deals` (deal_id, lead_id, customer_id, product_id, state, value, expected_close)
- `activities` (id, customer_id, type, channel, direction, summary, created_at)
- `pipelines` (snapshots for reporting: period, stage, count, value)

Common keys link out: `customer_id` (SCM/Accounting), `product_id` (SCM), `order_id` created on WON (Finance/Accounting). This is the integration point the Coordinator reads.

## 4. Deterministic tools (NO LLM)
- `capture_lead(source, payload)` — webhook from landing page / signup.
- `score_lead(lead_id)` — rule-based scoring (firmographics, source, engagement).
- `scrape_social(customer_id)` — pull mentions/interactions from configured social accounts → `activities` + sentiment count.
- `send_message(channel, customer_id, template_id, vars)` — WhatsApp/Telegram send. Returns delivery receipt.
- `recompute_forecast()` — roll pipeline value by stage/period.
- `transition_state(deal_id, to_state)` — validates legal transition, fires side-effects, logs.

LLM never touches these. Numbers come from the store.

## 5. Agent (Hermes harness) responsibilities
The agent runs inside the **Hermes harness**, which is the orchestration layer that drives tool-use, planning, and memory. LLM selection is **provider-agnostic via the Nous Portal** — MARU picks the model from Nous so the reasoning logic can improve over time as models evolve, with zero code rewiring.

- **Reporting:** daily/monthly CRM performance per fixed template (pipeline value, conversion by stage, lead sources, decay signals). Reads aggregates, formats — never computes.
- **Drafting:** outbound copy (greeting, promo, re-engagement) from templates + customer context.
- **Proposing:** when state signals decay (e.g. QUALIFIED stalled > N days, sentiment drop), propose re-engagement action → lands in `proposals` with evidence.
- **Tool-use loop:** inspects state → calls deterministic tools → observes results, all within bounds defined by the harness. The agent never holds a number it computed; it reads state back from the store.

## 6. Approval actions (outbound)
| Action            | Default mode | Notes                                  |
|-------------------|--------------|----------------------------------------|
| Send greeting     | Automated    | pre-whitelisted safe class (new signup)|
| Send promo        | Manual       | always human-confirm                   |
| Launch campaign   | Manual       | always human-confirm                   |
| Re-engagement     | Manual       | agent-proposed, evidence-attached      |

Nothing sends until `proposals.status = approved`.

## 7. Event bus + proposal flow
`transition_state` / `scrape_social` emit events → Coordinator (later) and Proposal Engine consume. In CRM-only phase, the Proposal Engine is local: rules + agent pattern detection → `proposals` row → console → human decides → `send_message` executes.

## 8. Reporting templates (sketch)
- **Daily:** new leads, qualified today, deals advanced, messages sent, decay alerts.
- **Monthly:** pipeline by stage, win-rate, source mix, forecast vs actual, top decay accounts.
- **Custom:** user asks "performance of X segment last Q" → agent formats from store.

## 9. Open questions for MARU + CRM domain owner (must resolve before coding)
1. Lead sources: which landing page / form? Single biz site or multiple? Webhook spec?
2. Social platforms to scrape: WhatsApp Business, Telegram, FB/IG, LinkedIn? Which API/provider?
3. Messaging provider: Twilio / 360dialog / direct Telegram Bot API? (drives `send_message`)
4. Greeting trigger: on signup only, or also on QUALIFIED?
5. Scoring rules: what defines a "qualified" lead for YOUR business?
6. CRM domain owner = who? (the user who signs off this module)
7. Forecast period: monthly? weekly?

## 10. Phase 1 build checklist (when approved)
- [ ] Phase 0 schema exists (common keys + audit_log + proposals + tuning + tax_config)
- [ ] `leads/contacts/accounts/deals/activities/pipelines` tables
- [ ] `capture_lead`, `score_lead`, `transition_state`, `recompute_forecast` tools
- [ ] `scrape_social`, `send_message` tools (provider wired)
- [ ] State-machine engine + audit logging
- [ ] Hermes harness reporting (daily/monthly templates) — Nous-Portal model selectable
- [ ] Approval console (proposals queue + Manual|Automated)
- [ ] Observability view (click state → source rows)
- [ ] Learning layer: `tuning` table + episodic capture in `audit_log` (§11)

## 11. Learning & tuning (agentic, observable)
The agent must **improve over time** — but never by silent weight-shifts. Two mechanisms, both stored:

1. **Episodic memory** — every agent action + human decision + outcome is already in `audit_log` (actor, payload, result). The agent queries its own history (e.g. "which greetings converted last quarter") from structured rows, not LLM recall.
2. **Learned policy (tuning)** — the proposal rules (when to suggest a re-engagement, when to flip a class to Automated) start as MARU's rules. The harness tracks which proposals were approved/rejected and the outcome that followed, then proposes a **tuning adjustment** as a row in the `tuning` table with evidence + who approved the change. A tuning change only takes effect after human sign-off.

This keeps the agent "agentic and learning" while every evolution is transparent and gated. The LLM reasons; the data proves; the human approves.

---
Drafted by Neo for discussion. No implementation until MARU + CRM domain owner confirm §9 answers and sign off.
