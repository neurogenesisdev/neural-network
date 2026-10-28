<%*
let title = tp.file.title;
if (title.startsWith("Untitled")) {
  title = (await tp.system.prompt("Project title")) || title;
  const slug = title.toLowerCase().replace(/[^a-z0-9äöüß]+/g, "-").replace(/^-+|-+$/g, "");
  if (slug) await tp.file.rename(slug);
}
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");
const today = moment().format("YYYY-MM-DD");
-%>
---
type: project
status: active
created: <% stamp %>
updated: <% stamp %>
area: "[[]]"
due:
tags: [project]
---

# <% title %>

## Goal
_What does "done" look like?_

## Success Criteria
- [ ]

## Tasks
### Next Actions
- [ ]

### Waiting For
- [ ]

## Resources
- [[]]

## Log
### <% today %>
-
