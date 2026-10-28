---
type: decision
id: ADR-
status: <% tp.system.suggester(["proposed", "accepted", "superseded", "deprecated"], ["proposed", "accepted", "superseded", "deprecated"]) %>
impact: <% tp.system.suggester(["low", "medium", "high"], ["low", "medium", "high"]) %>
provenance: <% tp.system.suggester(["measured", "inferred", "inherited-unverified"], ["measured", "inferred", "inherited-unverified"]) %>
device: <% tp.system.suggester(["arch-thinkpad", "netcup-vps", "pixel-graphene", "shared"], ["arch-thinkpad", "netcup-vps", "pixel-graphene", "shared"]) %>
constrains:
stale: false
created: <% tp.date.now("YYYY-MM-DDTHH:mm:ss") %>Z
updated: <% tp.date.now("YYYY-MM-DDTHH:mm:ss") %>Z
last_verified: <% tp.date.now("YYYY-MM-DD") %>
commit: pending
supersedes:
superseded_by:
tags: []
---

# <% tp.file.title %>

## Summary

One sentence. Enough for a reader — or an agent — to decide whether to read on.

## Context

What situation forced a choice. State the constraint that made this a decision
rather than a default.

## Decision

Imperative, one or two sentences.

## Alternatives Considered

1. **Alternative A** — why rejected.
2. **Alternative B** — why rejected.

## Consequences

### Positive

- 

### Negative

- 

## Revisit when

Condition or date. Delete the section if there is none.
