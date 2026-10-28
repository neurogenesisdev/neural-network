---
type: decision
id: FDR-001
aliases: [FDR-001]
status: accepted
impact: high
owner: mentor
client:
supersedes:
superseded_by:
created: 2026-09-09
updated: 2026-09-09
tags: [decision, hosting, ownership]
---

# No hosting on personal infrastructure

> [!NOTE] Summary
> Client sites and services are never hosted on Luis's own machines. Default is
> static hosting inside the client's own account.

## Context

The netcup-vps is Luis's personal, hardened server. Hosting a client there
makes him the permanent operator (backups, updates, "the site is down") and puts
the client's uptime on a box he pays for while funds are limited. On a pro-bono
project this converts a portfolio piece into unpaid maintenance.

## Decision

On 2026-09-08 Luis decided: client deliverables run in the client's own
accounts. For static sites the default is Cloudflare Pages in the client's
Cloudflare account, Luis added as a member. Any exception requires a paid
retainer and a written maintenance agreement.

## Alternatives considered

1. **Host on netcup-vps** — rejected: cost and ops burden land on Luis; no clean exit at handover.
2. **Luis's own Cloudflare account, client as member** — rejected: ownership inverted; handover becomes a migration.

## Consequences

- Positive: zero hosting cost, clean handover, no on-call obligation.
- Negative / cost: hosting is chosen per client from platforms with free or client-payable tiers; the Architect confirms fit per project.

## Revisit when

Luis offers a paid care plan that explicitly includes hosting.
