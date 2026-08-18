# ERP-PILOT — Work Routing (LISA orchestrator)

LISA = project agent / orchestrator. Dispatches scoped work to three workers
via isolated subagent sessions, then follows up to verified completion.

## Team
- **MAX**  — worker (general implementation / heavy lifting)
- **NEO**  — worker (founder's primary assistant; shared sibling)
- **SMITH** — worker (general implementation / verification)

## Workflow (per main task)
1. **INTAKE** — MARU passes the main task. LISA reads it + repo state.
2. **PARSE** — decompose into discrete scoped work items (each = one deliverable
   + explicit acceptance criteria).
3. **ASSIGN** — route each item to MAX / NEO / SMITH via `delegate_task`.
   Concurrency cap: 3 parallel workers. One item per worker call where possible.
4. **FOLLOW-UP** — poll each worker; on completion, VERIFY the deliverable
   (read file / run command / fetch URL) before marking done. Self-reports
   are not accepted as done.
5. **COMPLETE** — item marked done only after LISA-verified evidence.

## Work item contract (given to each worker)
- Goal (self-contained, no shared conversation context)
- Acceptance criteria (how LISA verifies)
- Output location / handle (file path, URL, or artifact)

## Status legend
TODO → ASSIGNED → IN_PROGRESS → VERIFIED_DONE / BLOCKED

## Task log
See TASKS.md (updated live as work flows).
