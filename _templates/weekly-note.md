<%*
// Weekly note. File name = ISO week, e.g. 2026-W40 (Periodic Notes format: GGGG-[W]WW).
const title = tp.file.title;
const parsed = moment(title, "GGGG-[W]WW", true);
const start = (parsed.isValid() ? parsed : moment()).clone().startOf("isoWeek");
const end = start.clone().add(6, "days");
const fmt = "GGGG-[W]WW";
const prev = start.clone().subtract(7, "days").format(fmt);
const next = start.clone().add(7, "days").format(fmt);
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");

// One embed per day: pulls each daily note's Review section into this note.
const dayLines = [];
for (let i = 0; i < 7; i++) {
  const day = start.clone().add(i, "days");
  const d = day.format("YYYY-MM-DD");
  const path = `journal/daily/${day.format("YYYY/MM")}/${d}`;
  dayLines.push(`### [[${path}|${d}]]\n![[${path}#Review]]`);
}
-%>
---
type: journal
status: draft
created: <% stamp %>
updated: <% stamp %>
week: <% start.format(fmt) %>
start: <% start.format("YYYY-MM-DD") %>
end: <% end.format("YYYY-MM-DD") %>
tags:
  - weekly-note
---
[[journal/weekly/<% prev %>|← Previous]] | <% start.format(fmt) %> | [[journal/weekly/<% next %>|Next →]]
# Week <% start.format("WW") %> · <% start.format("YYYY-MM-DD") %> – <% end.format("YYYY-MM-DD") %>

## Days
<% dayLines.join("\n\n") %>

---

## Review
<!-- Read the days above first. -->
- What moved forward:
- What stalled:
- What I learned:

---

## Promote
<!-- Ideas from this week's Logs that deserve their own note. Create the note, then link it here. -->
- 

---

## Plan
<!-- The 1–3 outcomes for next week. -->
1. 
