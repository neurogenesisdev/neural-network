<%*
// Blog post. Asks for a title when the note is still "Untitled" and renames the file to a slug.
let title = tp.file.title;
if (title.startsWith("Untitled")) {
  title = (await tp.system.prompt("Post title")) || title;
  const slug = title.toLowerCase().replace(/[^a-z0-9äöüß]+/g, "-").replace(/^-+|-+$/g, "");
  if (slug) await tp.file.rename(slug);
}
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");
-%>
---
type: blog-post
status: draft
created: <% stamp %>
updated: <% stamp %>
title: "<% title.replace(/"/g, '\\"') %>"
date: <% moment().format("YYYY-MM-DD") %>
excerpt: ""
tags: []
published: false
---

<% tp.file.cursor() %>
