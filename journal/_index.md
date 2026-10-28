---
type: index
status: active
created: 2026-04-21
updated: '2026-04-21T17:05:54Z'
tags: [index, journal]
sorting-spec: |
  target-folder: journal
  daily
  weekly
  monthly
  quarterly
  yearly
  %

  target-folder: journal/daily/*
  > a-z

  target-folder: journal/weekly
  target-folder: journal/monthly
  target-folder: journal/quarterly
  target-folder: journal/yearly
  > a-z
---

# journal — index

Dated journal entries: the daily notes and the periodic reviews built from them. Periodic Notes creates each one from its template.

## Subfolders

- `daily/` — one note per day at `daily/YYYY/MM/YYYY-MM-DD.md`, from [[daily-note]]. Post-2024 notes are active; the three 2021 reconstructions stay in place.
- [[journal/weekly/_index|weekly/]] — one review per ISO week, `GGGG-[W]WW.md`.
- [[journal/monthly/_index|monthly/]] — one review per month, `YYYY-MM.md`.
- [[journal/quarterly/_index|quarterly/]] — one review per quarter, `YYYY-[Q]Q.md`.
- [[journal/yearly/_index|yearly/]] — one review per year, `YYYY.md`.

## Files

- `tasks.md` — the task inbox. The home dashboard creates it on first use, for tasks whose due date has no daily note.

## File explorer order

The `sorting-spec` property of this note is read by the Custom File Explorer sorting plugin (custom-sort), which has this note set as its designated note. In `journal/` the order is daily, weekly, monthly, quarterly, yearly, then anything else. `daily/` with its year and month folders, and each review folder, list newest first. No other folder has a spec.
