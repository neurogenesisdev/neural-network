---
type: standard
status: active
created: 2026-09-09
updated: 2026-09-09
tags: [standard, scope, legal]
---

# Scope

> [!NOTE] Summary
> The twelve sections every scope document contains, and the rules for what
> may enter one.

Every scope document contains, in this order:

1. Goal — one primary outcome the deliverable exists for.
2. Deliverables — numbered, each testable.
3. Out of scope — explicit list, including everything the client mentioned that was deferred ("Shop — phase 2, paid").
4. Assumptions — and what happens if one is wrong.
5. Client responsibilities — content, images, logins, feedback deadlines.
6. Revision rounds — default 2; what counts as a round.
7. Timeline — milestones with dates.
8. Acceptance criteria — how both sides know it is done.
9. Ownership and handover — domain, hosting, accounts in the client's name.
10. Legal — Impressum, Datenschutzerklärung, cookie consent if embeds/tracking, AVV if Luis maintains systems with the client's customer data.
11. Maintenance option — or an explicit "none".
12. Price or trade.

Rules:
- Only `client` and `luis` facts go in. An `assumed` fact is a question, not a scope line.
- New requests after sign-off go to the out-of-scope list as a phase-2 item, never absorbed silently.
- The client confirms in writing before any build work starts.
