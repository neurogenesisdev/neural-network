<%*
let title = tp.file.title;
if (title.startsWith("Untitled")) {
  title = (await tp.system.prompt("Case title")) || title;
  const slug = title.toLowerCase().replace(/[^a-z0-9äöüß]+/g, "-").replace(/^-+|-+$/g, "");
  if (slug) await tp.file.rename(slug);
}
const today = moment().format("YYYY-MM-DD");
-%>
---
type: case
status: open
created: <% today %>
updated: <% today %>
counterparty:
aktenzeichen:
amount:
direction:            # owe | claim | none
deadline:
tags: [case]
---

# <% title %>

> [!NOTE] Summary
> One or two lines: who, what, how much, what happens next.

## Facts
- (letter/luis/assumed)

## Next action
- [ ]

## Waiting for
-

## Correspondence
- [[]]

## Log
### <% today %>
-
