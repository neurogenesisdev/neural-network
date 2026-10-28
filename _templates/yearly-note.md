<%*
// Yearly note. File name = YYYY, e.g. 2026 (Periodic Notes format: YYYY).
const title = tp.file.title;
const parsed = moment(title, "YYYY", true);
const start = (parsed.isValid() ? parsed : moment()).clone().startOf("year");
const year = start.format("YYYY");
const prev = start.clone().subtract(1, "year").format("YYYY");
const next = start.clone().add(1, "year").format("YYYY");
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");

// One embed per month: pulls each monthly note's Review section into this note.
const monthLines = [];
for (let i = 0; i < 12; i++) {
  const m = start.clone().add(i, "months").format("YYYY-MM");
  monthLines.push(`### [[journal/monthly/${m}|${m}]]\n![[journal/monthly/${m}#Review]]`);
}
-%>
---
type: journal
status: draft
created: <% stamp %>
updated: <% stamp %>
year: <% year %>
tags:
  - yearly-note
---
[[journal/yearly/<% prev %>|← Previous]] | <% year %> | [[journal/yearly/<% next %>|Next →]]
# <% year %>

## Resolutions
<!-- Written at the start of the year: the few things this year is for. -->
1. 

---

## Months
<% monthLines.join("\n\n") %>

---

## Review
<!-- Written at the end of the year. Read the months above first. -->
- What I'm proud of:
- What didn't happen, and why:
- What changed in how I live and work:
- Resolutions: kept, dropped, carried over:

---

## Plan
<!-- What carries into next year. Copy the keepers into next year's Resolutions. -->
1. 
