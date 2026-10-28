<%*
// Monthly note. File name = YYYY-MM, e.g. 2026-10 (Periodic Notes format: YYYY-MM).
const title = tp.file.title;
const parsed = moment(title, "YYYY-MM", true);
const start = (parsed.isValid() ? parsed : moment()).clone().startOf("month");
const end = start.clone().endOf("month");
const prev = start.clone().subtract(1, "month").format("YYYY-MM");
const next = start.clone().add(1, "month").format("YYYY-MM");
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");

// One embed per ISO week whose Monday falls in this month.
const weekLines = [];
let monday = start.clone().startOf("isoWeek");
if (monday.isBefore(start)) monday.add(7, "days");
while (!monday.isAfter(end)) {
  const w = monday.format("GGGG-[W]WW");
  weekLines.push(`### [[journal/weekly/${w}|${w}]]\n![[journal/weekly/${w}#Review]]`);
  monday.add(7, "days");
}
-%>
---
type: journal
status: draft
created: <% stamp %>
updated: <% stamp %>
month: <% start.format("YYYY-MM") %>
tags:
  - monthly-note
---
[[journal/monthly/<% prev %>|← Previous]] | <% start.format("YYYY-MM") %> | [[journal/monthly/<% next %>|Next →]]
# <% start.format("YYYY-MM") %>

## Weeks
<% weekLines.join("\n\n") %>

---

## Review
<!-- Read the weeks above first. -->
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
<!-- The 1–3 outcomes for next month. -->
1. 
