---
type: client
status: <% tp.system.suggester(["lead", "discovery", "scoping", "building", "review", "launched", "closed"], ["lead", "discovery", "scoping", "building", "review", "launched", "closed"]) %>
engagement: <% tp.system.suggester(["pro-bono", "fixed-price", "retainer", "hourly"], ["pro-bono", "fixed-price", "retainer", "hourly"]) %>
language: <% tp.system.suggester(["de", "en", "de+en"], ["de", "en", "de+en"]) %>
target_launch:
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
tags: [client]
---

# Client: <% tp.file.title %>

> [!NOTE] Summary
> Who, what they want, engagement type, current status — three lines.

## Who

Mark each fact `(client, date)`, `(luis)`, or `(assumed)`.

## Goal

What the deliverable must *do* for them in 12 months. One primary goal.

## Engagement terms

- Price / trade:
- Revision rounds: 2
- Third-party costs paid by: client, directly
- Ownership: all accounts on the client's email; Luis as member; credentials rotated at handover

## Scope

Link the scope document once agreed: [[scope]]. Until then: TBD.

## Timeline

| Milestone | Date | Status |
|-----------|------|--------|
| Discovery call | | |
| Scope agreed in writing | | |
| Design direction picked | | |
| Content delivered | | |
| Preview URL | | |
| Launch | | |
| Handover | | |

## Open questions (assumed facts)

- 

## Log

- <% tp.date.now("YYYY-MM-DD") %> — note created

## Related

- Contact: `contact.md` (gitignored)
- Tech: [[tech/_index]] (Architect)
