<%*
let title = tp.file.title;
if (title.startsWith("Untitled")) {
  title = (await tp.system.prompt("Map title")) || title;
  const slug = title.toLowerCase().replace(/[^a-z0-9äöüß]+/g, "-").replace(/^-+|-+$/g, "");
  if (slug) await tp.file.rename(slug);
}
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");
-%>
---
type: moc
status: active
created: <% stamp %>
updated: <% stamp %>
tags: [moc]
up: "[[home]]"
---

# <% title %>

> [!info] Purpose
> This MOC gathers notes about [topic].

## Core Concepts
- [[]]

## Related Ideas
- [[]]

## Questions to Explore
-

## Resources
- [[]]

---
**Related MOCs:** [[]]
