<%*
// Quarterly note. File name = YYYY-[Q]Q, e.g. 2026-Q4 (Periodic Notes format: YYYY-[Q]Q).
const title = tp.file.title;
const parsed = moment(title, "YYYY-[Q]Q", true);
const start = (parsed.isValid() ? parsed : moment()).clone().startOf("quarter");
const fmt = "YYYY-[Q]Q";
const prev = start.clone().subtract(1, "quarter").format(fmt);
const next = start.clone().add(1, "quarter").format(fmt);
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");

// One embed per month of this quarter: pulls each monthly note's Review section into this note.
const monthLines = [];
for (let i = 0; i < 3; i++) {
  const m = start.clone().add(i, "months").format("YYYY-MM");
  monthLines.push(`### [[journal/monthly/${m}|${m}]]\n![[journal/monthly/${m}#Review]]`);
}
-%>
---
type: journal
status: draft
created: <% stamp %>
updated: <% stamp %>
quarter: <% start.format(fmt) %>
tags:
  - quarterly-note
---
[[journal/quarterly/<% prev %>|← Previous]] | <% start.format(fmt) %> | [[journal/quarterly/<% next %>|Next →]]
# <% start.format(fmt) %>

## Months
<% monthLines.join("\n\n") %>

---

## Review
<!-- Read the months above first. -->
- What moved forward:
- What stalled:
- Habits to add, change or retire (edit the registry in [[habits-list]]):

---

## Maintenance
- Archive finished projects
- Note clusters of 6+ that need a map
- Unused tags to remove

---

## Plan
<!-- The 1–3 outcomes for next quarter. -->
1.
