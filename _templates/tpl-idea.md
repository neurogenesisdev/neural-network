<%*
// Idea note for projects/ideas/. The home dashboard's "New idea" button creates it from the
// capture field; the file name is the slug, the heading is the readable title.
const title = tp.file.title.replace(/-/g, " ").replace(/^./, (c) => c.toUpperCase());
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");
const category = (await tp.system.suggester(
  ["Build: code, automation, web", "Learn: a piece, a poem, a skill", "Go: a place or an event", "Join: a club, a class, a group"],
  ["build", "learn", "go", "join"])) ?? "";
-%>
---
type: idea
status: idea
category: <% category %>
effort:             # hour / evening / weekend / ongoing
projects: []        # related projects or notes, e.g. "[[web-presence]]"
created: <% stamp %>
updated: <% stamp %>
tags: []
---

# <% title %>

> [!NOTE] Summary
> <% tp.file.cursor() %>

## Context

Where it came from: a journal entry, a conversation, a site.

## First step

- 

## Notes

