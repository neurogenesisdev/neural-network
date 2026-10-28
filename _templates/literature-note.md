<%*
let title = tp.file.title;
if (title.startsWith("Untitled")) {
  title = (await tp.system.prompt("Source title")) || title;
  const slug = title.toLowerCase().replace(/[^a-z0-9äöüß]+/g, "-").replace(/^-+|-+$/g, "");
  if (slug) await tp.file.rename(slug);
}
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");
-%>
---
type: literature
status: unread
created: <% stamp %>
updated: <% stamp %>
author:
source:
tags: []
---

# <% title %>

## Summary
_Brief summary in your own words_

## Key Ideas
1.
2.
3.

## Quotes
> "Quote" (p. XX)

## Permanent Notes to Create
- [ ]

## Related
- [[]]
