---
type: decision
id: FDR-002
aliases: [FDR-002]
status: accepted
impact: medium
owner: mentor
client:
supersedes:
superseded_by:
created: 2026-09-09
updated: 2026-09-09
tags: [decision, pricing]
---

# Client pays third-party costs directly

> [!NOTE] Summary
> Domains, hosting, and subscriptions are bought by the client with the
> client's own payment method from day one. Luis never fronts costs.

## Context

Luis was willing to buy the first client's domain (~€12) and be reimbursed once
she was satisfied. The amount is trivial; the pattern is not: "Luis pays,
reimburse later" and "reimburse when satisfied" make a gift conditional and set
the expectation that Luis carries costs.

## Decision

On 2026-09-08 Luis decided: the client buys with their own card, in their own
account. Nothing in the build depends on the domain — a preview URL carries the
project until the last week.

## Alternatives considered

1. **Front and invoice later** — rejected: precedent, and on a free project there is no invoice.
2. **Buy early to "secure" a name** — rejected unless the client wants a specific name; then the client buys it early, themselves.

## Consequences

- Positive: clean ownership, no reimbursement conversation, no domain transfer later.
- Negative / cost: the client must complete a purchase before launch week; it goes on their to-do list early.
