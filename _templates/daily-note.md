<%*
// Daily note. The routine and the habits are generated from the registry in
// areas/self/habits-list.md (frontmatter `habits:`). Edit them there.
const title = tp.file.title;
const day = moment(title, "YYYY-MM-DD", true).isValid() ? moment(title, "YYYY-MM-DD") : moment();
const iso = day.format("YYYY-MM-DD");
// Daily notes live at journal/daily/YYYY/MM/YYYY-MM-DD.md; prev/next link by full path.
const dailyLink = (m) => `journal/daily/${m.format("YYYY/MM/YYYY-MM-DD")}`;
const prev = dailyLink(day.clone().subtract(1, "day"));
const next = dailyLink(day.clone().add(1, "day"));
const stamp = moment.utc().format("YYYY-MM-DDTHH:mm:ss[Z]");
const DOW = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"];
const dow = DOW[day.isoWeekday() - 1];

const toISO = (v) => v == null ? null
  : v.toISODate ? v.toISODate()
  : v instanceof Date ? v.toISOString().slice(0, 10)
  : String(v).slice(0, 10);

const regFile = app.vault.getAbstractFileByPath("areas/self/habits-list.md");
const reg = (regFile && app.metadataCache.getFileCache(regFile)?.frontmatter?.habits) || [];

// For each id keep the newest rule that applies on this day.
const active = new Map();
for (const h of reg) {
  if (!h || !h.id) continue;
  const since = toISO(h.since) ?? "0000-01-01";
  const until = toISO(h.until);
  if (since > iso || (until && until < iso)) continue;
  const cur = active.get(h.id);
  if (!cur || since >= cur.since) active.set(h.id, { ...h, since });
}

// Grouped by routine block, in registry order.
const lines = [];
let block = null;
for (const h of active.values()) {
  const fixedDays = Array.isArray(h.days) && h.days.length > 0;
  if (fixedDays && !h.days.includes(dow)) continue; // rest day: leave it out
  if (h.block && h.block !== block) {
    block = h.block;
    lines.push(`${lines.length ? "\n" : ""}### ${block}`);
  }
  const hint = Number(h.per_week) > 0 ? ` (${h.per_week}x/wk)` : h.optional ? " (optional)" : "";
  const min = h.minutes ? " [min:: ]" : "";
  lines.push(`- [ ] ${h.name ?? h.id}${hint} #habit [h::${h.id}]${min}`);
}
const habitLines = lines.length ? lines.join("\n") : "_No habits in the registry (areas/self/habits-list.md)._";
-%>
---
type: journal
status: draft
created: <% stamp %>
updated: <% stamp %>
date: <% iso %>
creation_time: <% moment().format("HH:mm:ss") %>
tags:
  - daily-note
work_shift:         # late / early / off
wake_time:          # HH:MM
bedtime:            # HH:MM, the night before
total_sleep_duration:   # hh:mm
rem_sleep_duration:     # hh:mm
deep_sleep_duration:    # hh:mm
mood_score:         # 1–5
energy_score:       # 1–5
focus_score:        # 1–5
run_type:           # easy / threshold / intervals / freeform / ruck
run_km:             # e.g. 7.5
training_order:     # e.g. iso-run-strength
first_meal:         # HH:MM
last_meal:          # HH:MM
late_work:          # true / false: freelance work after the shift
logged:             # set by the home dashboard when a habit is ticked there
focus_theme:        # short one-liner (e.g., "Deep Work – Coding")
---
[[<% prev %>|← Previous]] | <% iso %> | [[<% next %>|Next →]]
# <% day.format("dddd, Do MMMM, YYYY") %>

## ✅ Habits
<% habitLines %>

---

## Tasks


---

# 📝 Journal
## Plan
<!-- Morning: the 1–3 outcomes that would make today a good day. -->
1. 

## Log
<!-- During the day: ideas, observations, what you worked on. Link out to notes instead of writing topics here. -->
- 

## Review
<!-- Evening: what went well, what didn't, what to carry into tomorrow. -->
- 

