---
type: lesson
severity: <% tp.system.suggester(["critical", "important", "minor"], ["critical", "important", "minor"]) %>
client:
changed:
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
tags: [lesson]
---

# <% tp.file.title %>

> [!NOTE] Summary
> What happened and the rule it produced, in two lines.

## What happened

Facts, dated.

## What it cost

Hours, money, goodwill, or a delayed milestone. Quantify.

## Rule extracted

One imperative line. Copy it into the standard named in `changed`
(e.g. `changed: standards/delivery.md`). If no standard changes, this is not a
lesson — delete the note.

## Template changed

Which file in `templates/` was edited as a result, if any.
