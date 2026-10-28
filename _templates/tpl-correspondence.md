<%*
let title = tp.file.title;
if (title.startsWith("Untitled")) {
  title = (await tp.system.prompt("Correspondence title")) || title;
  const slug = title.toLowerCase().replace(/[^a-z0-9äöüß]+/g, "-").replace(/^-+|-+$/g, "");
  if (slug) await tp.file.rename(slug);
}
const today = moment().format("YYYY-MM-DD");
-%>
---
type: correspondence
status: open           # open | answered | filed
created: <% today %>
updated: <% today %>
received: <% today %>
doc_date:
sender:
channel:               # post | email | portal
kind:                  # bescheid | mahnung | inkasso | rechnung | info | antrag | antwort
amount:
deadline:
aktenzeichen:
case: "[[]]"
attachment: "[[_attachments/letters/]]"
tags: [correspondence]
---

# <% title %>

> [!NOTE] Summary
> What the document says and what it asks for, in two lines.

## Key data
- Sender:
- Reference:
- Amount:
- Deadline:
- Payment details: (IBAN / Verwendungszweck — only here, never in an index)

## Required response
- [ ]

## Notes
-
