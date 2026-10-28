---
type: backlog-item
status: <% tp.system.suggester(["active", "blocked", "done", "archived"], ["active", "blocked", "done", "archived"]) %>
priority: <% tp.system.suggester(["high", "medium", "low"], ["high", "medium", "low"]) %>
device: <% tp.system.suggester(["arch-thinkpad", "netcup-vps", "pixel-graphene", "shared"], ["arch-thinkpad", "netcup-vps", "pixel-graphene", "shared"]) %>
projects: []
tags: []
created: <% tp.date.now("YYYY-MM-DDTHH:mm:ss") %>Z
updated: <% tp.date.now("YYYY-MM-DDTHH:mm:ss") %>Z
commit: pending
---

# <% tp.file.title %>

> [!NOTE] Summary
> One paragraph. What is undone, and what it costs to leave it undone. Enough
> for a reader — or an agent — to decide whether to open the note.

## Context

Why this exists. Where it came from. If it was inherited from an older
document, say which, and whether the original framing still holds.

## Steps

1. 
2. 

## Blocked on

Delete this section if nothing blocks it. If something does, name the specific
condition that would unblock it, not a vague dependency.

## Note

Constraints, traps, and the step most likely to be skipped. This is where the
value is — the steps are usually obvious, the trap is not.

## Related

- [[]]
