---
type: reference
status: active
created: 2026-10-22
tags: [system, dashboard]
---

# Dashboard library

Shared code for the Datacore blocks on [[home]]. They load a section of this note with `dc.require(dc.headerLink("_meta/dashboard-lib.md", "<section>"))`. Datacore runs the first JavaScript code block under the heading.

## Daily note paths

Daily notes live at `journal/daily/YYYY/MM/YYYY-MM-DD.md`; change the layout here and nowhere else.

```js
const DAILY = "journal/daily";
// The year and month folders must match the date in the file name.
const DAY_PATH = /^journal\/daily\/(\d{4})\/(\d{2})\/(\1-\2-\d{2})\.md$/;

// "2026-10-22" -> "journal/daily/2026/10/2026-10-22.md"
const dailyPath = (iso) => `${DAILY}/${iso.slice(0, 4)}/${iso.slice(5, 7)}/${iso}.md`;

// A daily note's path -> its day ("2026-10-22"); null for anything else (reviews, journal/tasks.md, stray files).
const dayOf = (path) => {
    const m = String(path ?? "").match(DAY_PATH);
    return m ? m[3] : null;
};

return { DAILY, dailyPath, dayOf };
```
