# ERP-PILOT · BFS ERP — Task Log

Status: ACTIVE (mobile prototype phase)

## Direction (locked with MARU)
- App: BFS ERP — web app, smartphone-friendly first.
- Phase: PROTOTYPE (front-end only, fake data, no backend).
- Shape: multi-screen mobile app, bottom nav, tappable interactions.
- Replaces the single-scroll overview (index.html kept as desktop overview only).

## Routing model
LISA = orchestrator/assembler. Workers run as isolated parallel subagents
(delegate_task), each owns 1–2 screens, returns file paths + verification.
LISA verifies every file, then assembles app.html shell.

## Workers → Screens
- NEO   → screen-01 (SSO/Profile) + screen-02 (WhatsApp Mileage)
- MAX   → screen-03 (Events Scraper) + screen-04 (Booking)
- SMITH → screen-05 (Packages/Pricing)

## Items
| id | item | owner | status |
|----|------|-------|--------|
| 1 | app.html shell: phone frame + top bar + bottom nav + screen switch | LISA | pending |
| 2 | screen-01 SSO/Profile (tap Google → fake logged-in profile + demo tiles) | NEO | in_progress |
| 3 | screen-02 WhatsApp Mileage (oil-due + re-engage bubbles, mileage gauge) | NEO | in_progress |
| 4 | screen-03 Events (scan→cards→blast, tappable) | MAX | in_progress |
| 5 | screen-04 Booking (branch/slot grid, tap slot→confirm) | MAX | in_progress |
| 6 | screen-05 Packages (cards, tap Book→sheet) | SMITH | in_progress |
| 7 | LISA assemble app.html + verify (nav maps, theme, responsive) | LISA | pending |
| 8 | Commit mobile prototype | LISA | pending |
