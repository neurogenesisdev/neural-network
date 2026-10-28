<%*
// Permanent (concept) note. Asks for a title when the note is still "Untitled" and renames the file to a slug.
let title = tp.file.title;
if (title.startsWith("Untitled")) {
  title = (await tp.system.prompt("Concept: one idea, as a short title")) || title;
  const slug = title.toLowerCase().replace(/[^a-z0-9äöüß]+/g, "-").replace(/^-+|-+$/g, "");
  if (slug) await tp.file.rename(slug);
}
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");
-%>
---
type: permanent
status: active
created: <% stamp %>
updated: <% stamp %>
tags: []
aliases: []
up: "[[]]"
---

# <% title %>

## Idea
(One atomic idea in your own words)

<% tp.file.cursor() %>

## Evidence
- [[Source]]

## Related
- [[]]

---
*Links to:* [[MOC Name]]
