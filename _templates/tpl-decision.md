---
type: decision
id: FDR-
status: <% tp.system.suggester(["proposed", "accepted", "superseded"], ["proposed", "accepted", "superseded"]) %>
impact: <% tp.system.suggester(["low", "medium", "high"], ["low", "medium", "high"]) %>
owner: <% tp.system.suggester(["mentor", "architect", "shared"], ["mentor", "architect", "shared"]) %>
client:
supersedes:
superseded_by:
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
tags: [decision]
---

# <% tp.file.title %>

> [!NOTE] Summary
> One sentence. Enough to decide whether to read on.

## Context

What forced a choice. Name the constraint — time budget, money, legal, ownership.

## Decision

Past tense, dated. "On YYYY-MM-DD Luis decided to …"

## Alternatives considered

1. **A** — why rejected.
2. **B** — why rejected.

## Consequences

- Positive:
- Negative / cost:

## Revisit when

Condition or date. Delete if none.
