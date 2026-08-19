# ERP-PILOT

> Agentic ERP for SMEs — deterministic business engines wrapped by an agentic AI harness, with a human always at the approval gate.

ERP-PILOT is a from-scratch, open-source ERP built for small and medium enterprises. It is **not** a chatbot bolted onto a database. It is a set of **deterministic business engines** (one per module) that compute, transact, and act through tooling — wrapped by an **agentic AI harness** (Hermes) that plans, drafts, reports, and proposes. Every irreversible action is proposed by the system and confirmed by a human.

---

## Why this exists

Most SME "ERP" tools are spreadsheets that draw dashboards. Existing open-source ERPs (Odoo, ERPNext, Dolibarr, SuiteCRM, Tryton) are AGPL/GPL — forking them forces you to open-source your product, and none of them have the agentic coordinator or the deterministic-engine-plus-LLM-reporting split this project is built on.

We are building bottom-up because the value is in the **process/state backbone** and the **shared ontology**, not in a form that collects data. A module here is a *state machine with triggers*, not a table.

## Core architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      HUMAN APPROVAL CONSOLE                   │
│         (dashboard · approval queue · observability)          │
└───────────────▲───────────────────────────┬─────────────────┘
                │ approve/reject             │ configure
┌───────────────┴───────────┐   ┌────────────▼──────────────────┐
│   AGENTIC HARNESS (Hermes)│   │   COORDINATOR (synthesis)      │
│  - goal → plan → tool-use │   │  - cross-module read          │
│  - draft outbound copy    │   │  - rule synthesis + report    │
│  - episodic + policy learn│   │  - optional proposals         │
└───────────────▲───────────┘   └────────────▲──────────────────┘
                │ calls tools                 │ reads/writes
┌───────────────┴─────────────────────────────┴──────────────────┐
│              DETERMINISTIC TOOL / LOGIC LAYER                   │
│   Every number, transaction, scrape, send = pure function/API  │
│   LLM NEVER computes a figure or executes an irreversible act  │
└───────────────────────────▲───────────────────────────────────┘
                            │ writes structured state
┌───────────────────────────┴───────────────────────────────────┐
│              SHARED STATE  (Supabase, one ontology)            │
│   common keys: customer_id · product_id · order_id · ledger_id │
│   + audit_log · proposals · tax_config · tuning               │
└────────────────────────────────────────────────────────────────┘
```

**Three layers, strict boundaries:**

1. **Deterministic Tool/Logic Layer** — all calculation, all transactions, all integrations. No LLM. SST tax, ledger postings, WhatsApp sends, social scrapes — every one is a tool with a typed input/output and a logged result.
2. **Agentic Harness (Hermes)** — the LLM operates *inside* a harness: it receives a goal, inspects shared state, selects and calls tools, observes outcomes, and drafts human-facing output (reports, greetings, proposals). It plans and proposes; it does not compute money or execute sends.
3. **Human Approval Console** — every outbound/irreversible action becomes a `proposals` row first. Nothing executes until a human approves. Per action type, the owner can pre-set **Manual** (always ask) or **Automated** (pre-whitelisted safe classes only).

## The agent harness — what "agentic" means here

The harness is not a single prompt. It is:

- **Tool-use:** the agent calls deterministic tools; it never trusts a number it "remembers" — it reads state back from the store.
- **Planning loop:** given a goal (e.g. *improve this week's lead conversion*), the agent inspects state, picks tools, acts within bounds, observes results, iterates.
- **Learning over time (two kinds, both observable):**
  - *Episodic memory* — every agent action + outcome is logged to the store. The agent later queries "what worked last quarter" from structured history, not from LLM memory.
  - *Learned policy* — proposal rules start as **your** rules. The harness tracks which proposals were approved/rejected and what outcome followed, then tunes future proposals. A "learned adjustment" is a **row in a tuning table** with its evidence and who approved the change. Never a silent weight shift.
- **Provider-agnostic by design:** the harness runs on **Hermes**, so the LLM is selected from the Nous Portal provider list. As models improve, you swap the provider without rewiring logic. Dev on the current free model; production can pin a stronger one.

**Boundary we hold regardless of "agentic":** the harness may plan, propose, and draft — but it never computes financial/SST figures and never executes irreversible actions. Those stay deterministic tools behind the approval gate.

## Outbound model — AI triggers, human confirms

1. **Event bus** — the tool layer emits state changes (new lead, invoice overdue, stock low, churn signal).
2. **Proposal engine** — deterministic rules fire proposals; the harness may *also* propose from pattern detection (e.g. *sentiment dropped → suggest campaign*). Every proposal carries the evidence that triggered it.
3. **Approval console** — human approves/rejects. On approve → tool executes. Two modes per action type:
   - **Manual:** always asks.
   - **Automated:** owner pre-whitelists safe categories (e.g. greeting new signups auto-sends; promos always need approval).

Nothing irreversible executes without a human gate in v1. Every proposal and decision is written to `audit_log`.

## Transparency principle

ERP-PILOT v0 was rejected because it hid the backend and had no process pipeline. This build mandates observability: every module exposes its state, each process step, and each tool action in a view. The Coordinator's synthesis traces back to source rows — if it says *pause procurement*, you click through to the CRM-leads-down and Finance-runway-tight data that drove it. **No black boxes.**

## Tech stack

| Concern        | Choice                                             |
|----------------|----------------------------------------------------|
| Database/state | New dedicated Supabase project (Postgres + RLS)    |
| Runtime/engine | Supabase Edge Functions + cron (event handlers)     |
| UI/console     | Next.js on a new dedicated Vercel project           |
| Agent harness  | Hermes (LLM via Nous Portal, provider-selectable)  |
| Deploy         | Vercel auto-deploy on push to `main`               |

> Separate Supabase + Vercel from RINA and other projects. Clean isolation.

## Shared data layer / ontology

The Coordinator can only "share valuable data" if every module writes to **one structured store with common keys**. Because tools (not agents) write the data, it is already clean. The ontology is the single integration point and is defined in **Phase 0** before any module is built.

Common keys: `customer_id`, `product_id`, `order_id`, `ledger_id`.
Cross-cutting tables: `audit_log`, `proposals`, `tax_config` (versioned SST rates), `tuning` (learned policy adjustments).

---

## Module roadmap (sequential — one proven before the next starts)

> No parallel module work. Each module is built as a vertical slice: state machine → deterministic tools → harness integration → reporting → approval actions.

### 1. CRM  *(first vertical slice)*
- **Purpose:** turn leads into revenue through a *process*, not a contact list.
- **Process / state machine:** `Lead → Qualified → Proposal → Negotiation → Won/Lost`. Each transition is an event firing tool actions (notify owner, scrape account social, draft re-engagement WhatsApp, update forecast).
- **Deterministic tools:** landing-page capture webhook, signup webhook, social-media scrape + mention/sentiment count, WhatsApp/Telegram send, forecast recompute.
- **Agent (Hermes) responsibilities:** summarize daily/monthly performance per fixed template; draft outbound copy (greetings, promos); propose re-engagement when state signals decay.
- **Approval actions:** send greeting (auto-ok class), send promo (manual), launch campaign (manual).

### 2. Accounting — Malaysian SST  *(second vertical, deterministic-critical)*
- **Purpose:** compliant SST computation and ledger, zero LLM in the math.
- **Deterministic engine:** chart of accounts (SME-typical) + `tax_config` (sales tax 5%/10%, service tax 6%/8%, zero-rated, exempt). Every transaction tagged with a tax code → engine computes output tax, claimable input tax, net SST payable per period. Inclusive/exclusive handling, input-tax credit, periodic return figure. Rates live in a **versioned config table** (budget changes) — auditable, updatable without code changes.
- **Agent responsibilities:** report *"Q3 SST payable = RM X, input claimable = RM Y"* — never computes it.
- **Approval actions:** post journal (manual), file return (manual).

### 3. Finance
- **Purpose:** cashflow, runway, budgeting on top of the Accounting ledger.
- **Deterministic tools:** cashflow projection, runway calc, budget variance.
- **Agent responsibilities:** synthesize financial health reports; propose spend holds when runway tight.
- **Approval actions:** transfer (manual), freeze spend (manual).

### 4. SCM (Supply Chain Management)
- **Purpose:** stock, suppliers, fulfillment.
- **Process:** `PO → Received → Stocked → Picked → Shipped`. Stock-cover alerts.
- **Deterministic tools:** stock levels, reorder calc, fulfillment status.
- **Agent:** propose reorder; report stock health.
- **Approval actions:** raise PO (manual), expedite (manual).

### 5. Procurement
- **Purpose:** sourcing, PO issuance, vendor management (overlaps SCM at the buy side).
- **Deterministic tools:** vendor comparison, PO generation, spend analysis.
- **Agent:** recommend vendor; draft RFQ.
- **Approval actions:** issue PO (manual), approve vendor (manual).

### 6. Marketing
- **Purpose:** campaigns, content, channel performance.
- **Deterministic tools:** campaign tracking, channel metrics, A/B results.
- **Agent:** draft content, propose campaign from CRM/Finance signal.
- **Approval actions:** publish post (manual), launch ad (manual).

### 7. Change Management
- **Purpose:** track org/process changes, adoption, risk.
- **Deterministic tools:** change-request state machine, impact log, training tracking.
- **Agent:** summarize change health; flag resistance patterns.
- **Approval actions:** approve change (manual).

---

## Coordinator

Because modules are deterministic and write shared state, the Coordinator is a **rule engine + reporter**, not a free-thinking planner:

- Runs on schedule (daily) or on-demand.
- Reads cross-module aggregates (CRM pipeline value, Finance runway, SCM stock cover, Accounting SST exposure).
- Applies **your** business rules (e.g. *if pipeline < 3mo burn AND stock cover > 60d → flag promo + pause procurement*).
- Emits a Coordinator Report (LLM-formatted) + optional Proposals into the approval queue.

Built as a thin stub after two modules exist, grown as more come online.

---

## Phase plan

- **Phase 0 — Foundation:** shared schema + ontology + `audit_log` + `proposals` + `tax_config` (SST) + `tuning`. No agents yet.
- **Phase 1 — CRM vertical slice:** proves the whole pattern (engine + state machine + harness + approval + reporting).
- **Phase 2 — Accounting (SST):** proves the deterministic-calculation pattern.
- **Phase 3–7 — Finance, SCM, Procurement, Marketing, Change-Mgmt** in sequence.
- **Coordinator** layered in once ≥2 modules exist.

---

## Developer protocol (co-dev)

This repo is built **module by module, discussed before coded**.

**For every module, before any code is written, the assigned developer (human or agent) MUST:**

1. Open a discussion with **MARU (founder)** and the **user** owning that module's domain.
2. Confirm: the module's process/state-machine, its deterministic tools, its agent responsibilities, its approval actions, and its reporting templates.
3. Write the module spec as a markdown file under `docs/modules/<module>.md`.
4. Only after MARU + the domain user sign off does implementation start.

**Injected instruction for agent-assisted development:** any agent coding a module must first present its understanding of the module's process, tools, agent scope, and approval actions, and explicitly ask MARU and the domain user to confirm or correct it **before writing implementation code**. No silent assumptions.

---

## Status

- [x] Vision & architecture defined
- [x] README / spec (this document)
- [ ] Phase 0 — shared schema + ontology
- [ ] Phase 1 — CRM vertical slice
- [ ] Phase 2 — Accounting (SST)
- [ ] Phases 3–7 — remaining modules
- [ ] Coordinator

---

## License

MIT — see [LICENSE](./LICENSE). Free to use, modify, and redistribute, including commercially. (Contributors: keep changes open per the co-dev protocol above.)
