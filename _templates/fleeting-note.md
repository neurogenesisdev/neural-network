<%*
let title = tp.file.title;
if (title.startsWith("Untitled")) {
  title = (await tp.system.prompt("Fleeting note title")) || title;
  const slug = title.toLowerCase().replace(/[^a-z0-9äöüß]+/g, "-").replace(/^-+|-+$/g, "");
  if (slug) await tp.file.rename(slug);
}
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");
-%>
---
type: fleeting
status: active
created: <% stamp %>
updated: <% stamp %>
tags: []
---

# <% title %>

## Capture
<% tp.file.cursor() %>

## Source
- Where did this come from?

## Next
- [ ] Process into permanent note or discard
